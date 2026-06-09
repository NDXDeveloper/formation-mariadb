🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 19.6 Tests de compatibilité

La section précédente (§19.5) a recensé *ce qu'il faut vérifier* lors d'une migration : protocole, pilotes, dialecte SQL, sémantique d'exécution, authentification, collations. La présente section traite de la question complémentaire : *comment le vérifier concrètement*, avec quelle méthodologie et quel outillage, avant de basculer en production.

Le principe directeur est économique : le coût de détection d'une incompatibilité croît de plusieurs ordres de grandeur entre l'environnement de test et la production. Une régression trouvée par un test automatisé coûte quelques minutes ; la même régression découverte par un utilisateur en production coûte un incident, une investigation à chaud, et potentiellement un *rollback* (§19.7). L'objectif des tests de compatibilité est donc de **faire remonter en amont, et de façon reproductible, les écarts de comportement** — en particulier les incompatibilités silencieuses, qui ne produisent aucune erreur mais un résultat différent.

---

## 19.6.1 Une stratégie de test à plusieurs niveaux

Tester une migration ne se réduit pas à lancer la suite de tests de l'application. Une stratégie complète couvre plusieurs niveaux, du plus structurel au plus comportemental :

| Niveau | Objet du test | Question vérifiée |
|--------|---------------|-------------------|
| **Schéma** | Structure des objets | La structure est-elle reproduite à l'identique ? |
| **Données** | Contenu des tables | Les données sont-elles cohérentes source vs cible ? |
| **Fonctionnel** | Comportement applicatif | L'application produit-elle les mêmes résultats ? |
| **Requêtes** | Résultats SQL | Une requête donnée renvoie-t-elle la même chose ? |
| **Performance** | Temps de réponse, plans | Les performances sont-elles maintenues ? |
| **Comportement 12.3** | Changements de version | Les nouveautés de comportement sont-elles gérées ? |

Ces niveaux ne sont pas redondants : une migration peut réussir le test de schéma et de données (la structure et le contenu sont identiques) tout en échouant au niveau comportemental (une requête trie différemment à cause d'une collation). C'est précisément cette dissociation qui justifie une approche multi-niveaux.

---

## 19.6.2 Environnements de test

La fiabilité des tests dépend directement de la **représentativité de l'environnement**. Tester contre une instance vide ou avec quelques lignes de données ne détecte ni les problèmes de performance, ni les écarts de plan d'exécution, qui ne se manifestent qu'à l'échelle réelle.

Un environnement de validation efficace présente trois caractéristiques :

**Une instance MariaDB 12.3 configurée comme la cible de production.** Même version mineure, même `sql_mode`, mêmes variables (en tenant compte des variables retirées — §11.2.3), mêmes plugins d'authentification. Une configuration de test qui diffère de la cible invalide les conclusions. La conteneurisation (§16.3) facilite la reproduction exacte de cette configuration.

**Un volume de données représentatif.** Idéalement une copie récente de la production, à défaut un jeu de données dimensionné de façon réaliste. Le volume conditionne les plans d'exécution choisis par l'optimiseur : un plan optimal sur 1 000 lignes peut être catastrophique sur 10 millions.

**Des données anonymisées si elles proviennent de la production.** La copie de données réelles vers un environnement de test soulève des enjeux de protection des données personnelles (RGPD). Les données sensibles doivent être pseudonymisées ou anonymisées avant utilisation hors production.

Le schéma classique est une **infrastructure parallèle** (*side-by-side*) : l'ancienne instance (11.8 ou MySQL) et la nouvelle (12.3) coexistent, alimentées par les mêmes données, ce qui permet les comparaisons directes décrites plus bas.

---

## 19.6.3 Validation du schéma

La première vérification consiste à s'assurer que la **structure** des objets (tables, index, contraintes, vues, routines, triggers, événements) a été reproduite fidèlement. Une extraction du schéma seul, sans données, permet une comparaison textuelle :

```bash
# Extraire la structure seule, sur chaque instance
mariadb-dump --no-data --skip-comments --routines --events --triggers \
  ma_base > schema_source.sql

mariadb-dump --no-data --skip-comments --routines --events --triggers \
  --host=cible-12.3 ma_base > schema_cible.sql

# Comparer
diff schema_source.sql schema_cible.sql
```

Les écarts attendus doivent être interprétés, pas seulement constatés : un changement de collation par défaut (utf8mb4 / UCA 14.0.0, §11.11) ou de scope des noms de contraintes de clés étrangères (uniques par table en 12.3, §18.12) apparaîtra légitimement dans le diff. L'enjeu est de distinguer ces différences *attendues* d'éventuelles différences *non voulues* (un index manquant, une contrainte non reprise).

Pour une validation plus structurée, l'interrogation d'`INFORMATION_SCHEMA` permet de comparer programmatiquement le nombre d'objets, les types de colonnes ou les définitions d'index entre les deux bases (§9.7.1). Les outils de gestion des migrations de schéma comme Flyway et Liquibase (§16.8) intègrent par ailleurs des mécanismes de validation de l'état du schéma par rapport à un historique de versions.

Enfin, après une montée de version in-place, l'exécution de **`mariadb-upgrade`** (§19.4.1) met à jour les tables système et signale les objets nécessitant une attention, tandis que **`CHECK TABLE`** (§11.6.3) valide l'intégrité physique des tables, y compris désormais les SEQUENCE.

---

## 19.6.4 Tests de régression fonctionnelle

Le niveau fonctionnel consiste à **exécuter la suite de tests de l'application** (tests unitaires, d'intégration, de bout en bout) en la pointant vers l'instance MariaDB 12.3. C'est le filet de sécurité le plus direct : si l'application dispose d'une couverture de tests sérieuse, une part importante des incompatibilités fonctionnelles y sera capturée.

La valeur de cette approche est strictement proportionnelle à la **couverture des tests**. Un parcours non couvert par un test ne sera pas validé. Avant une migration, il est donc utile d'identifier les chemins critiques (transactions financières, traitements par lots, requêtes de reporting) et de s'assurer qu'ils sont effectivement exercés par la suite de tests.

Cette exécution doit se faire avec la configuration cible exacte — en particulier le `sql_mode` (§11.3) et le niveau d'isolation par défaut — afin que les tests reproduisent les conditions de production et non un environnement permissif.

---

## 19.6.5 Validation de la cohérence des données

Après une migration de données, il faut prouver que le contenu de la cible correspond à celui de la source. Plusieurs niveaux de rigueur sont possibles.

Le plus rapide est la comparaison des **comptages de lignes** par table, qui détecte les pertes ou duplications grossières mais ne dit rien du contenu cellule par cellule.

L'instruction native **`CHECKSUM TABLE`** calcule une empreinte du contenu d'une table, comparable entre deux instances :

```sql
-- Sur chaque instance, comparer les empreintes
CHECKSUM TABLE commandes, clients, produits;
```

Pour une validation plus fine et adaptée aux grandes tables, **`pt-table-checksum`** (Percona Toolkit) découpe les tables en segments et calcule des empreintes par segment, ce qui permet d'identifier *quelles lignes* diffèrent et non seulement *qu'il existe* une différence. L'outil complémentaire `pt-table-sync` peut ensuite réconcilier les écarts détectés.

> ⚠️ Attention aux faux positifs : une différence de collation peut faire apparaître des empreintes divergentes alors que les données « logiques » sont identiques, simplement triées ou normalisées différemment. L'interprétation des écarts doit tenir compte des changements de comportement attendus (§19.5.4).

---

## 19.6.6 Comparaison automatisée des résultats de requêtes

C'est l'outil le plus directement adapté à la validation d'une montée de version : **`pt-upgrade`** (Percona Toolkit) exécute un même ensemble de requêtes contre deux instances et **compare les résultats, les avertissements et les erreurs**. C'est exactement la réponse au risque d'incompatibilité silencieuse identifié en §19.5.4.

Le corpus de requêtes peut provenir du slow query log, du general log ou des binary logs — c'est-à-dire d'un trafic réel capturé sur la production :

```bash
# Comparer les résultats des requêtes du slow log entre l'ancien serveur et 12.3
pt-upgrade \
  h=ancien-serveur,P=3306 \
  h=cible-12.3,P=3306 \
  --user=app_test --password=*** \
  slow-query.log
```

`pt-upgrade` signale notamment : les requêtes dont le **jeu de résultats diffère** entre les deux serveurs, les requêtes qui **réussissent sur l'une et échouent sur l'autre**, et les différences d'avertissements. Un cas utile à détecter lorsqu'on migre **depuis MySQL ou une MariaDB antérieure à 11.6.2** : une requête transactionnelle qui réussissait sur l'ancien système et se heurte désormais à une erreur de conflit d'instantané (`innodb_snapshot_isolation`, §6.9). Entre 11.8 et 12.3, en revanche, ce comportement est inchangé (le paramètre est déjà à `ON` en 11.8).

> 🔗 `pt-upgrade` appartient à la même boîte à outils que `pt-query-digest` (§15.7.2), utilisé pour l'analyse des requêtes lentes ; les deux se nourrissent des mêmes journaux capturés en production.

---

## 19.6.7 Capture et rejeu de charge de production

Au-delà des requêtes isolées, l'idéal est de rejouer un **trafic de production réaliste** contre la nouvelle version, avec sa concurrence, son volume et ses motifs d'accès. MaxScale fournit pour cela une chaîne native (§14.5) :

- **Workload Capture** (§14.5.1) enregistre le trafic réel arrivant sur la production.
- **Workload Replay** (§14.5.2) rejoue ce trafic capturé contre l'instance MariaDB 12.3, reproduisant la charge concurrente.
- **Diff Router** (§14.5.3) envoie *les mêmes requêtes* aux deux backends (ancien et nouveau) et **compare les réponses en temps réel**, signalant les divergences.

Cette approche est particulièrement précieuse pour deux raisons. D'abord, elle exerce des chemins que les tests synthétiques omettent souvent (requêtes ad hoc, combinaisons rares). Ensuite, le Diff Router automatise la comparaison à grande échelle : plutôt que d'inspecter manuellement des milliers de réponses, l'opérateur reçoit la liste des requêtes dont le résultat diverge — c'est-à-dire la liste des incompatibilités silencieuses à investiguer.

La combinaison `pt-upgrade` (sur un corpus de logs) et MaxScale Workload Capture/Replay + Diff Router (sur un trafic vivant) couvre de façon complémentaire la comparaison de résultats : l'un travaille hors ligne sur un échantillon, l'autre en conditions de charge.

---

## 19.6.8 Tests de non-régression de performance et comparaison des plans

Une migration peut conserver des résultats *fonctionnellement* identiques tout en dégradant les performances. Deux raisons en 12.3 :

- l'**optimiseur basé sur les coûts** prend désormais en compte les caractéristiques SSD (§15.14) et bénéficie de nouvelles optimisations (scans inversés, §5.8.1) ;
- les **Optimizer Hints** (§15.15) modifient le contrôle du plan d'exécution.

Conséquence : pour une même requête, MariaDB 12.3 peut choisir un **plan d'exécution différent** de la 11.8 — généralement meilleur, parfois en régression sur un cas particulier. La comparaison des plans est donc un test de compatibilité à part entière, et pas seulement un test de performance.

La démarche consiste à capturer le plan des requêtes critiques sur les deux versions et à comparer :

```sql
-- Sur chaque instance, pour chaque requête critique
EXPLAIN FORMAT=JSON
SELECT ... ;            -- comparer les plans

ANALYZE
SELECT ... ;            -- exécution réelle (forme MariaDB ; équivaut à EXPLAIN ANALYZE de MySQL)
```

Les écarts à surveiller : un index utilisé sur une version et ignoré sur l'autre, un changement de type de jointure, l'apparition d'un tri ou d'une table temporaire. Lorsqu'une régression est confirmée, les Optimizer Hints (§15.15) permettent de forcer le plan souhaité.

Au niveau macroscopique, un **benchmark comparatif** avec sysbench ou mysqlslap (§15.12) mesure le débit et la latence sous charge sur les deux versions, fournissant un indicateur global de non-régression.

---

## 19.6.9 Tests ciblés des comportements spécifiques à la 12.3

En complément des tests génériques, certains changements de comportement de la 12.3 (recensés en §19.5.6) appellent des scénarios de test dédiés, car ils ne seront pas nécessairement déclenchés par une suite de tests existante :

**Isolation par instantané** (§6.9). Un scénario de deux transactions concurrentes modifiant la même ligne sous `REPEATABLE READ` permet de vérifier que l'application capture bien l'erreur de conflit et applique sa logique de rejeu, plutôt que de la propager à l'utilisateur. Ce test est surtout déterminant pour une application migrée **depuis MySQL ou une MariaDB antérieure à 11.6.2** : pour un passage `11.8 → 12.3`, le comportement est identique des deux côtés (paramètre déjà à `ON` en 11.8), mais le scénario reste un bon contrôle de robustesse.

**Collations utf8mb4 / UCA 14.0.0** (§11.11). Un jeu de données combinant casse, accents et caractères sur 4 octets (émojis) permet de valider les tris (`ORDER BY`), les égalités (`WHERE col = ...`) et surtout les contraintes `UNIQUE`, dont le comportement dépend directement de la collation.

**`sql_mode` strict** (§11.3). Des insertions aux valeurs limites (chaîne trop longue, date hors plage, type incompatible) permettent de vérifier que l'application gère correctement les rejets sous le mode strict de la cible.

**Variables retirées** (§11.2.3). Un simple test de démarrage (*smoke test*) avec la configuration de production cible vérifie qu'aucun fichier de configuration ni script d'initialisation ne référence `big_tables`, `large_page_size` ou `storage_engine`.

**Nouveau format de binlog** (§11.5.4). Si une chaîne de capture de changements (CDC, Debezium — §20.8) consomme le binlog, un test d'intégration vérifie qu'elle prend en charge le nouveau format intégré à InnoDB.

---

## 19.6.10 Connectivité, authentification et automatisation

**Tests de connectivité et d'authentification.** Chaque combinaison connecteur × méthode d'authentification utilisée par les applications doit être validée explicitement : un test de connexion minimal par pilote (§19.5.2) et par plugin d'authentification (§19.5.5) confirme qu'aucune application ne sera bloquée au démarrage. Ce point est rapide à tester mais critique, car un échec d'authentification empêche toute autre vérification.

**Intégration dans le pipeline CI/CD.** Pour rendre ces tests reproductibles et systématiques, ils gagnent à être intégrés à la chaîne d'intégration continue (§16.7). Une instance MariaDB 12.3 conteneurisée (§16.3) est instanciée à chaque exécution du pipeline, la suite de tests fonctionnels et les vérifications de schéma (Flyway/Liquibase, §16.8) s'y exécutent automatiquement, et tout écart bloque la livraison. Cette automatisation transforme la compatibilité d'un audit ponctuel en une garantie continue, utile bien au-delà de la migration initiale.

---

## 19.6.11 Critères de validation (go / no-go)

Tester sans critères d'acceptation prédéfinis expose à une décision de bascule subjective, prise sous la pression du calendrier. Avant de lancer la campagne de tests, il est sain de définir les conditions de réussite — par exemple : aucune divergence de résultat non expliquée signalée par `pt-upgrade` ou le Diff Router, suite de tests fonctionnels intégralement au vert, cohérence des données confirmée, absence de régression de performance au-delà d'un seuil convenu, et connexion validée pour chaque application.

Ces critères matérialisent la frontière entre une migration prête et une migration à reporter. Ils alimentent directement la décision de bascule et, le cas échéant, le déclenchement du plan de contingence décrit dans la section suivante (§19.7).

---

## Points clés à retenir

- Les tests de compatibilité s'organisent sur **plusieurs niveaux** (schéma, données, fonctionnel, requêtes, performance, comportements 12.3) qui ne sont pas redondants : réussir l'un ne garantit pas les autres.
- La **représentativité de l'environnement** (version, configuration, volume de données) conditionne la validité des conclusions ; un volume réaliste est indispensable pour détecter les régressions de plan d'exécution.
- **`pt-upgrade`** et le **Diff Router de MaxScale** (§14.5) sont les outils centraux pour débusquer les incompatibilités *silencieuses*, en comparant les résultats d'une même requête sur deux versions.
- La comparaison des **plans d'exécution** (`EXPLAIN`) est un test de compatibilité à part entière, car l'optimiseur 12.3 (conscient du SSD, §15.14) peut choisir des plans différents.
- Certains comportements à risque lors d'une migration (snapshot isolation — pour les sources antérieures à 11.6.2 ou MySQL —, collations, `sql_mode` strict, variables retirées) appellent des **scénarios de test dédiés** qu'une suite existante ne déclenche pas spontanément.
- Intégrer ces tests au **pipeline CI/CD** (§16.7) transforme la compatibilité en garantie continue, et définir des **critères go/no-go** objective la décision de bascule.

> 🔗 **Pour aller plus loin** : §19.5 *Compatibilité des applications* (ce qu'il faut vérifier), §19.7 *Rollback et contingence* (que faire si les tests révèlent un problème après bascule), §14.5 *MaxScale : Workload Capture/Replay et Diff Router*, §15.12 *Benchmarking*, §5.7 *Analyse des plans d'exécution*, §16.7 *CI/CD pour bases de données*, et §19.10 *Migration 11.8 → 12.3* pour la liste complète des changements de comportement à couvrir.

⏭️ [Rollback et contingence](/19-migration-compatibilite/07-rollback-contingence.md)

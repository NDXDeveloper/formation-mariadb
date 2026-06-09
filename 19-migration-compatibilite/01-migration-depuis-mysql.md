🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 19.1 Migration depuis MySQL

> **Chapitre 19 — Migration et Compatibilité** · Section 19.1  
> Niveau : DBA / Architecte

---

## Le scénario de migration le plus courant

De toutes les migrations vers MariaDB, le passage depuis **MySQL** est de loin le plus fréquent et, généralement, le plus simple à conduire. Les deux moteurs partagent une origine commune, un protocole réseau largement compatible et un socle SQL très proche. Dans de nombreux cas, une application conçue pour MySQL fonctionne avec MariaDB moyennant des ajustements limités, parfois sans aucune modification du code applicatif.

Cette proximité ne doit cependant pas faire croire que la migration est triviale. Au fil des années, les deux produits ont suivi des trajectoires de plus en plus distinctes, et l'arrivée de **MySQL 8** a creusé l'écart sur plusieurs points structurants : authentification par défaut, dictionnaire de données, fonctions géospatiales, gestion du JSON, ou encore disparition du query cache. La compatibilité reste forte, mais elle n'est plus totale : elle doit être vérifiée, et non présumée.

L'objectif de cette section est de poser le cadre général de la migration depuis MySQL — son contexte historique, la nature de la compatibilité entre les deux moteurs et la démarche d'ensemble — avant d'entrer dans le détail des points techniques, des différences à surveiller et de l'outillage, traités dans les sous-sections suivantes.

---

## Un héritage commun, une divergence assumée

MariaDB est issue d'un **fork de MySQL** réalisé en 2009, dans le contexte du rachat de Sun Microsystems — alors propriétaire de MySQL — par Oracle. Michael « Monty » Widenius, cofondateur de MySQL, a lancé ce projet pour garantir la continuité d'une version libre et ouverte du moteur. Le nom suit d'ailleurs la même logique que celui de MySQL : tous deux sont nommés d'après les enfants de leur créateur. La gouvernance du projet est aujourd'hui assurée par la **MariaDB Foundation**, qui veille à son caractère ouvert.

Pendant ses premières années, MariaDB se voulait un **remplacement direct** (*drop-in replacement*) de MySQL : les numéros de version étaient alignés (MariaDB 5.1, 5.2, 5.3, 5.5) et la compatibilité binaire et fonctionnelle était quasi parfaite. C'est à partir de MariaDB 10.0 que le projet a adopté son propre schéma de versions et commencé à diverger franchement, en introduisant des fonctionnalités absentes de MySQL.

De son côté, MySQL a poursuivi son évolution propre, jusqu'à la rupture marquée par **MySQL 8**, qui a apporté des changements architecturaux profonds. Certaines fonctionnalités modernes (CTE, *window functions*, JSON) ont été ajoutées par les deux moteurs, mais souvent de façon indépendante, avec des implémentations et des comportements qui ne coïncident pas. La conséquence pratique est claire : **MariaDB n'est plus présentée comme un remplacement direct de MySQL pour les versions récentes**. La migration depuis MySQL 8 reste tout à fait réalisable, mais elle relève désormais d'un véritable projet de migration, et non d'une simple substitution de binaires.

---

## Ce que recouvre la « compatibilité »

Parler de compatibilité entre MySQL et MariaDB recouvre en réalité plusieurs niveaux distincts, qu'il importe de ne pas confondre.

**Au niveau du protocole et des connecteurs**, la compatibilité est élevée : la plupart des pilotes et bibliothèques clientes (PHP, Python, Java, Node.js, Go, .NET) se connectent indifféremment à l'un ou l'autre moteur. Les exceptions concernent surtout les mécanismes d'authentification récents, abordés dans la sous-section 19.1.1.

**Au niveau du dialecte SQL**, le tronc commun est très large — la grande majorité des requêtes d'une application MySQL s'exécutent telles quelles sur MariaDB. Les divergences portent sur des fonctions spécifiques, des comportements implicites, certains types de données et des extensions propres à chaque moteur. Ces points font l'objet de la sous-section 19.1.2.

Il faut enfin souligner que la compatibilité n'est **pas symétrique**. Migrer de MySQL vers MariaDB est un chemin balisé et bien outillé ; le trajet inverse, de MariaDB vers MySQL, est nettement plus délicat, car MariaDB possède des fonctionnalités qui n'ont pas d'équivalent direct chez MySQL. Ce chapitre traite exclusivement du sens MySQL → MariaDB.

---

## L'importance de la version source

La difficulté d'une migration depuis MySQL dépend fortement de la **version de départ** :

Une migration **depuis MySQL 5.7 ou une version antérieure** est généralement la plus fluide. Ces versions sont historiquement très proches des éditions correspondantes de MariaDB, et les écarts à traiter restent limités.

Une migration **depuis MySQL 8 (ou ultérieur)** demande davantage d'attention. Les changements introduits par MySQL 8 — nouveau plugin d'authentification par défaut, refonte du support géospatial, évolutions du dictionnaire de données et du format JSON — constituent les principaux points de friction à analyser avant la bascule. C'est précisément le cas le plus courant aujourd'hui, et celui sur lequel la sous-section 19.1.1 met l'accent.

Identifier précisément la version source, ainsi que les fonctionnalités MySQL effectivement utilisées par l'application, constitue donc la toute première étape d'un audit de migration sérieux.

---

## Pourquoi migrer vers MariaDB

Les motivations d'une migration depuis MySQL sont variées et relèvent autant de considérations techniques que stratégiques. Sans prétendre à l'exhaustivité, on retrouve fréquemment :

- **La gouvernance et le modèle de licence.** MySQL est développé par Oracle selon un modèle à double licence (libre et commerciale). Certaines organisations privilégient le développement entièrement ouvert de MariaDB, encadré par la MariaDB Foundation.
- **Les fonctionnalités propres à MariaDB.** Le moteur propose des capacités absentes de MySQL ou implémentées différemment : moteurs de stockage additionnels (Aria, ColumnStore, Spider), tables temporelles à versionnement système, recherche vectorielle, couches de compatibilité Oracle, entre autres.
- **La continuité depuis une base MySQL existante.** Pour un parc déjà sous MySQL, MariaDB représente le chemin de migration le moins coûteux en raison de l'héritage commun.

Ces éléments sont présentés ici à titre de contexte : le choix d'une migration relève d'une analyse propre à chaque organisation, tenant compte de ses contraintes d'exploitation, de son écosystème et de ses besoins applicatifs.

---

## Démarche générale

Quelle que soit la version source, une migration depuis MySQL suit une trame en plusieurs étapes, qui sera précisée et outillée dans les sous-sections suivantes ainsi que dans le reste du chapitre :

1. **Audit préalable** : inventaire des versions, des schémas, des fonctionnalités MySQL utilisées et des dépendances applicatives.
2. **Analyse de compatibilité** : identification des points de friction (authentification, fonctions, types, comportements) — voir 19.1.1 et 19.1.2.
3. **Choix de la méthode** : migration logique (export/import), physique, ou via réplication, selon le volume de données et la tolérance à l'interruption.
4. **Tests** : validation sur un environnement représentatif, incluant les tests applicatifs et de non-régression (voir 19.6).
5. **Bascule (cutover)** : exécution de la migration, idéalement avec un plan de retour arrière (voir 19.7), et le cas échéant sans interruption de service (voir 19.8).
6. **Validation post-migration** : vérification de l'intégrité des données, des performances et du comportement applicatif.

---

## Organisation de la section

Les sous-sections suivantes approfondissent chacun de ces aspects pour le cas spécifique de MySQL :

- **19.1.1 — Compatibilité MySQL/MariaDB** : les points techniques clés, notamment l'authentification (`caching_sha2_password`) et les nouvelles fonctions géospatiales (GIS) introduites par MySQL 8.
- **19.1.2 — Points d'attention et différences** : les écarts de comportement, de fonctions et de types à anticiper lors de la bascule.
- **19.1.3 — Outils de migration** : l'outillage disponible pour préparer, exécuter et valider la migration.

⏭️ [Compatibilité MySQL/MariaDB (caching_sha2_password, nouvelles fonctions GIS MySQL 8)](/19-migration-compatibilite/01.1-compatibilite-mysql-mariadb.md)

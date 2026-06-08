🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 17.4 Bonnes pratiques de développement

Les sections précédentes ont montré **comment** se connecter à MariaDB et l'interroger ; celle-ci rassemble les principes qui font la différence entre un code qui « marche » et un code **fiable, performant et maintenable** en production. Plusieurs aspects ont leur propre section et ne sont ici que cités : migrations (§17.5), tests (§17.6), environnements (§17.7), injections SQL (§17.8) et *prepared statements* (§17.9).

---

## Gérer les connexions avec rigueur

Une connexion empruntée doit **toujours** être restituée, même en cas d'erreur. Chaque écosystème offre un mécanisme de fermeture automatique, à utiliser systématiquement : `try-with-resources` (Java, §17.1.3), `using`/`await using` (.NET, §17.1.6), `defer` (Go, §17.1.5), gestionnaires de contexte (Python, §17.1.2), `release()`/`end()` (Node.js, §17.1.4). Une connexion oubliée est une **fuite** qui finit par épuiser le pool (§17.2).

En production, on passe par un **pool** (§17.2) — jamais une connexion par requête — et l'on **n'ouvre pas plusieurs pools** sans raison. On valide les connexions (*pre-ping*) et l'on cale leur durée de vie sous le `wait_timeout` du serveur pour éviter les connexions périmées.

---

## Maîtriser les transactions

Une transaction doit être **courte et explicite**. Quelques règles :

- **La garder brève** : ne jamais maintenir une transaction ouverte pendant une opération lente (appel réseau externe, traitement de fichier) ou pendant un temps de réflexion utilisateur — elle retiendrait des verrous et nuirait à la concurrence (chap. 6).
- **Encadrer le `commit`/`rollback`** par une gestion d'erreurs : annuler en cas d'exception, valider seulement au succès.
- **Connaître le comportement d'autocommit** de son pilote, qui varie : désactivé par défaut côté Python (§17.1.2), activé côté Java, Go et .NET. En découle l'oubli classique du `commit` en Python, ou l'inverse ailleurs.
- **Choisir le bon niveau d'isolation** selon le besoin (chap. 6.3), sans systématiquement viser le plus strict.

---

## Écrire des requêtes efficaces

C'est là que se jouent la plupart des problèmes de performance applicative :

- **Éviter le N+1** — le piège n°1 des ORM (§17.3) : utiliser le chargement anticipé (`JOIN FETCH`, `include`, `selectinload`, `Include`) plutôt que d'accéder à une relation paresseuse dans une boucle.
- **Ne sélectionner que le nécessaire** — proscrire `SELECT *` ; ne demander que les colonnes utilisées, ce qui allège le transfert et favorise les index couvrants (chap. 5.9).
- **Paginer** — borner les résultats avec `LIMIT`/`OFFSET` ou une pagination par clé (*keyset*), au lieu de charger des jeux de résultats entiers en mémoire.
- **S'appuyer sur les index** — vérifier que les colonnes filtrées et jointes sont indexées (chap. 5).
- **Traiter par lots** — préférer une insertion/mise à jour groupée (`batch`, *bulk*) à une boucle de requêtes unitaires (cf. `batch` du connecteur `mariadb`, §17.1.4).
- **Diffuser les gros volumes** — pour de très grands résultats, parcourir en flux (curseurs côté serveur) plutôt que tout matérialiser.

---

## Observer le SQL réellement exécuté

Le fil rouge de tout ce chapitre : **on ne corrige que ce qu'on observe**. L'ORM ou le pilote masque le SQL ; il faut le rendre visible. En développement, activer la journalisation des requêtes (`show_sql` Hibernate, `echo=True` SQLAlchemy, `logging` Sequelize/Prisma, journalisation EF Core). En analyse, recourir à **`EXPLAIN`** (chap. 5.7) pour comprendre un plan d'exécution, et au **journal des requêtes lentes** (chap. 11.4.2, 15.7) pour repérer les coupables en production. C'est ainsi que l'on détecte un N+1, un balayage de table ou une requête inattendue.

---

## Paramétrer et sécuriser

Trois réflexes, approfondis ailleurs mais incontournables ici :

- **Toujours paramétrer** les requêtes (*prepared statements*, §17.9) — jamais de concaténation de valeurs dans le SQL, qui ouvre la porte aux injections (§17.8).
- **Moindre privilège** — le compte applicatif ne doit disposer que des droits strictement nécessaires (chap. 10), pas d'un accès administrateur.
- **Aucun secret en dur** — chaînes de connexion et mots de passe vivent dans la configuration, des variables d'environnement ou un gestionnaire de secrets (§17.7), jamais dans le code ni dans le dépôt.

---

## Gérer les erreurs intelligemment

Une couche d'accès aux données robuste distingue les erreurs **transitoires** des erreurs **permanentes** :

- les erreurs transitoires — **interblocage** (*deadlock*, chap. 6.5), expiration d'attente de verrou, perte de connexion, saturation temporaire — peuvent justifier une **nouvelle tentative** (avec un léger délai croissant). Un interblocage, en particulier, se résout en **rejouant la transaction**, pas en la considérant comme un bug ;
- les erreurs permanentes (violation de contrainte, syntaxe) doivent **remonter** clairement.

Dans tous les cas, ne jamais **avaler** silencieusement une erreur, et ne pas **exposer** de détails techniques sensibles (structure, requêtes) dans les messages renvoyés à l'utilisateur.

---

## Soigner types, encodage et fuseaux

Plusieurs pièges déjà rencontrés méritent une vigilance constante :

- **`utf8mb4` partout** — pour couvrir l'intégralité d'Unicode (§17.1, chap. 11.11).
- **Valeurs NULL** — les traiter explicitement (types *nullable* dédiés, §17.1.5) plutôt que de les ignorer.
- **Grands entiers** — attention à la précision des `BIGINT` au-delà de 2⁵³−1 dans les langages à flottants (§17.1.4).
- **Montants** — utiliser `DECIMAL` (et non `FLOAT`/`DOUBLE`) pour éviter les erreurs d'arrondi.
- **Temps** — raisonner en **UTC** côté serveur et gérer le fuseau à l'affichage, surtout en contexte distribué.

---

## Isoler l'accès aux données

Plutôt que de disséminer du SQL dans toute l'application, on **regroupe** l'accès aux données dans une couche dédiée (patron *repository* / *data access layer*). Cela clarifie le code, centralise les optimisations, et rend l'accès aux données **testable** de façon isolée — un préalable aux tests (§17.6).

---

## Gérer la concurrence

Quand plusieurs processus modifient les mêmes données, le **verrouillage optimiste** est souvent préférable au verrouillage pessimiste : une **colonne de version** est comparée lors de la mise à jour (`UPDATE … WHERE id = ? AND version = ?`) ; si aucune ligne n'est affectée, c'est qu'un autre a modifié l'enregistrement entre-temps, et l'on traite le conflit. La plupart des ORM le prennent en charge nativement (annotation de version). Pour l'historisation, on peut s'appuyer sur les **tables temporelles** (chap. 18.2).

---

## Ce qu'il faut retenir

- **Fermer/restituer** systématiquement les connexions (mécanismes automatiques de chaque langage) et passer par un **pool** (§17.2).
- **Transactions courtes et explicites**, conscient du **comportement d'autocommit** du pilote, isolation adaptée (chap. 6).
- **Requêtes efficaces** : éviter le **N+1** (§17.3), proscrire `SELECT *`, paginer, indexer (chap. 5), traiter par lots.
- **Observer le SQL** généré (journalisation, `EXPLAIN`, journal des requêtes lentes) — on ne corrige que ce qu'on voit.
- **Toujours paramétrer** (§17.8, §17.9), appliquer le **moindre privilège** (chap. 10), **aucun secret en dur** (§17.7).
- Distinguer erreurs **transitoires** (rejouer un *deadlock*) et **permanentes** ; soigner **types, `utf8mb4`, `DECIMAL`, UTC**.
- **Isoler l'accès aux données** (testabilité, §17.6) et gérer la **concurrence** (verrouillage optimiste).

⏭️ [Gestion des migrations de schéma](/17-integration-developpement/05-gestion-migrations-schema.md)

🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 17.6 Tests de bases de données

La couche d'accès aux données est critique et pourtant souvent sous-testée. Une requête erronée, un mapping incorrect, une migration défaillante, une transaction mal bornée : autant de bugs qui ne se révèlent qu'en production s'ils ne sont pas couverts par des tests. Cette section traite de la manière de **tester le code qui touche à la base**, en s'appuyant sur l'isolation de l'accès aux données (§17.4) et en renvoyant au chap. 16.3 (Docker) et 16.7 (CI/CD) pour l'infrastructure.

---

## Les niveaux de test

Le classique « pyramide des tests » s'applique au code base de données :

- **Tests unitaires** — ils vérifient la **logique métier** en isolant la couche d'accès aux données (que l'on remplace par un *mock*/*stub*). Rapides, sans base réelle. Mais ils **ne testent pas le SQL** : ni les requêtes, ni les mappings, ni les migrations.
- **Tests d'intégration** — ils exercent le **véritable accès aux données contre une base réelle**. Plus lents, mais ce sont eux qui détectent les vrais problèmes : correction du SQL, mappings, contraintes, transactions, migrations. C'est là que se concentre la valeur des tests de base de données.

L'isolation de l'accès aux données (patron *repository*, §17.4) est ce qui rend la logique métier testable unitairement **et** l'accès aux données testable séparément.

---

## Mock ou vraie base ?

La tentation est de tout *mocker* pour aller vite. C'est pertinent pour tester la logique métier au-dessus du *repository*, mais **inopérant pour valider le SQL lui-même** : un *mock* renverra ce qu'on lui dit, jamais ce que MariaDB renverrait réellement. *Mocker* le pilote ou la connexion est encore pire — fragile et sans valeur. Pour le code d'accès aux données, **il faut une base réelle** au niveau intégration. Reste à savoir laquelle et comment la gérer.

---

## Tester contre MariaDB, pas un substitut

C'est le principe le plus important de cette section. Par souci de rapidité, on est tenté de tester contre une base **en mémoire ou différente** : SQLite, H2 (Java) ou le *provider* « InMemory » d'EF Core. **C'est un anti-modèle.** Ces substituts n'ont ni le même dialecte, ni les mêmes types, ni les mêmes contraintes que MariaDB ; ils donnent une **fausse confiance** et laissent passer des erreurs qui n'apparaîtront qu'en production. Le *provider* InMemory d'EF Core, en particulier, n'est pas une base relationnelle et n'applique pas la sémantique relationnelle.

La règle : **tester contre le même moteur que la production — MariaDB —, idéalement la même version** (12.3).

---

## Obtenir une base de test jetable : Testcontainers

L'approche moderne de référence est **Testcontainers** : une bibliothèque qui démarre par programmation un conteneur MariaDB **jetable** le temps des tests, puis le détruit. Reproductible, isolé, identique en local et en CI, disponible pour Java, .NET, Node.js, Python et Go. Elle s'appuie sur Docker (chap. 16.3).

```java
@Testcontainers
class ProduitRepositoryIT {

    @Container
    static MariaDBContainer<?> mariadb = new MariaDBContainer<>("mariadb:12.3");

    // La connexion de test pointe vers mariadb.getJdbcUrl(), getUsername(), getPassword()
}
```

L'exemple est en Java, mais des équivalents existent dans chaque écosystème. D'autres options coexistent : un service MariaDB partagé via Docker Compose, ou une base/un schéma de test dédié — au prix d'une isolation moindre.

---

## Isoler les données entre les tests

C'est la partie délicate : chaque test doit partir d'un **état connu**, sans dépendre des précédents. Plusieurs stratégies, avec leurs compromis :

- **Rollback de transaction** — chaque test s'exécute dans une transaction annulée à la fin. Très rapide et propre, mais **inadapté** si le code testé valide lui-même (`commit`) ou gère ses propres transactions, et incapable de tester le comportement de `commit`.
- **Nettoyage entre tests** — vider (`TRUNCATE`/`DELETE`) les tables avant ou après chaque test. Des outils l'automatisent en respectant l'ordre des clés étrangères, comme **Respawn** (.NET) ou des pilotes de test comme **go-txdb** (Go).
- **Recréation du schéma** — la plus sûre mais la plus lente.
- **Jeux de données (*fixtures*)** — charger des données connues et minimales avant chaque test.

On combine souvent un schéma créé une fois (par les migrations) et un nettoyage rapide entre les tests.

---

## Tester les migrations

Les migrations sont du code : elles méritent d'être testées. En CI, on les **applique sur une base neuve** pour vérifier qu'elles s'exécutent sans erreur et produisent le **schéma attendu** ; si l'on pratique le *rollback*, on le teste aussi (§17.5). C'est la meilleure parade contre une migration qui « passe en local » mais casse ailleurs.

---

## Tester transactions, contraintes et concurrence

Certains comportements ne se vérifient **que** contre une vraie base : l'**annulation** d'une transaction en cas d'erreur, le déclenchement des **violations de contrainte** (clé étrangère, unicité), la gestion des **interblocages** (rejeu, §17.4/chap. 6.5) et le **verrouillage optimiste** (§17.4). Les couvrir suppose donc des tests d'intégration sur MariaDB.

---

## Par écosystème

Pour mémoire, en s'appuyant sur les pilotes du §17.1 :

- **Java** — JUnit + Testcontainers ; tranches de test Spring (`@DataJpaTest`, `@Sql`). Éviter H2 comme substitut.
- **Python** — pytest + Testcontainers ; *fixtures* et rollback de session SQLAlchemy.
- **Node.js** — Jest/Vitest + Testcontainers ; rollback de transaction.
- **.NET** — xUnit + Testcontainers ; **Respawn** pour réinitialiser l'état. Éviter le *provider* InMemory comme substitut.
- **Go** — `testing` + Testcontainers ; `go-txdb` pour l'isolation par transaction.

---

## Intégration continue

Les tests d'intégration ont toute leur place en CI : on y fournit MariaDB soit via un **service** du pipeline (conteneur de service GitHub Actions, GitLab CI…), soit via **Testcontainers**. Exécuter ces tests à chaque changement — y compris l'application des migrations — est ce qui transforme la couche d'accès aux données en composant **fiable** (chap. 16.7).

---

## Ce qu'il faut retenir

- Les **tests unitaires** (logique métier, *repository* simulé) ne testent pas le SQL ; les **tests d'intégration** sur base réelle, si — c'est là qu'est la valeur.
- **Tester contre MariaDB** (même version que la prod), **pas** un substitut en mémoire (SQLite, H2, EF InMemory) : ces derniers donnent une fausse confiance.
- Utiliser **Testcontainers** pour une base MariaDB **jetable et reproductible** (chap. 16.3), en local comme en CI.
- **Isoler les données** entre tests (rollback de transaction, nettoyage type Respawn/go-txdb, recréation, *fixtures*) — chaque test part d'un état connu.
- **Tester les migrations** sur une base neuve (§17.5), ainsi que transactions, contraintes et concurrence (§17.4, chap. 6.5).
- Exécuter les tests d'intégration en **CI** (chap. 16.7) ; isoler l'accès aux données rend l'ensemble testable (§17.4).

⏭️ [Environnements de développement](/17-integration-developpement/07-environnements-developpement.md)

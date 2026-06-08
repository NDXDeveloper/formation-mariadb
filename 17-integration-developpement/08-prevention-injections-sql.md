🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 17.8 Prévention des injections SQL

L'injection SQL est la vulnérabilité applicative la plus connue et l'une des plus graves : elle figure de longue date au sommet des classements de risques (OWASP). Tout au long de ce chapitre, les sections ont renvoyé ici. Le moment est venu de **comprendre la menace pour mieux la neutraliser**, puis d'exposer une stratégie de défense en couches. La parade principale — les requêtes paramétrées — est détaillée au §17.9 ; sa syntaxe par langage figure aux §17.1.x.

---

## Qu'est-ce qu'une injection SQL ?

Une injection SQL survient lorsqu'une **entrée non fiable** (saisie utilisateur, paramètre d'URL, en-tête, contenu importé…) est **concaténée dans une instruction SQL**. L'attaquant peut alors glisser des fragments de SQL dans ce qui devait être une simple valeur, et ainsi **altérer la structure et le sens** de la requête. La cause profonde est toujours la même : on **mélange le code (la requête) et les données (les valeurs)**.

---

## Le mécanisme

Considérons un code qui construit une requête en concaténant une saisie :

```text
// ❌ DANGEREUX — ne jamais faire cela
sql = "SELECT * FROM utilisateurs WHERE nom = '" + nomSaisi + "'"
```

Si l'utilisateur saisit une valeur normale, tout va bien. Mais s'il saisit `' OR '1'='1`, la requête devient :

```sql
SELECT * FROM utilisateurs WHERE nom = '' OR '1'='1'
```

La condition est toujours vraie : la requête renvoie **toutes les lignes**, contournant le filtre — un classique du contournement d'authentification. À partir de cette faille, un attaquant peut **lire des données qu'il ne devrait pas voir, en modifier, en détruire**, voire, dans les cas graves, compromettre le serveur. (Les requêtes empilées sont souvent désactivées par défaut selon le pilote, mais bien d'autres vecteurs subsistent : la concaténation reste dangereuse.)

---

## La parade principale : les requêtes paramétrées

La solution n'est pas d'échapper « mieux » les entrées, mais de **séparer le code des données**. Avec une requête **paramétrée** (*prepared statement*), la structure de la requête est fixée à l'avance, et les valeurs sont transmises **séparément** : elles ne sont **jamais interprétées comme du SQL**, quel que soit leur contenu.

```text
// ✅ SÛR — la valeur est liée, jamais interprétée comme du SQL
sql = "SELECT * FROM utilisateurs WHERE nom = ?"
// puis on lie nomSaisi en tant que paramètre
```

Saisir `' OR '1'='1` ne change alors rien à la structure : la chaîne est traitée comme une simple valeur recherchée. C'est **la** défense de référence, à appliquer **systématiquement**. Comme souligné dès le §17.1.1, le critère décisif n'est ni l'outil ni l'ORM, mais l'**usage systématique du paramétrage**. Le mécanisme est détaillé au §17.9.

---

## Défense en profondeur

Aucune couche unique ne suffit ; on les empile :

- **ORM et *query builders*** — ils paramètrent **par défaut** (§17.3), ce qui élimine l'essentiel du risque. Attention toutefois aux **portes de sortie vers le SQL brut** (`$queryRawUnsafe` de Prisma, `FromSqlRaw` d'EF Core, `sequelize.query`…) : on doit y paramétrer tout autant.
- **Les identifiants ne sont pas des paramètres** — un nom de table, de colonne, le sens d'un `ORDER BY` ou `ASC`/`DESC` **ne peuvent pas être liés** comme des valeurs. Pour un identifiant dynamique, on utilise une **liste blanche** stricte qui mappe la saisie vers un ensemble fixe de valeurs connues :

  ```text
  // ❌ DANGEREUX : ORDER BY construit depuis une saisie brute
  "... ORDER BY " + colonneSaisie

  // ✅ Liste blanche : seules des colonnes connues sont possibles
  autorisees = { "nom": "nom", "prix": "prix", "date": "cree_le" }
  colonne = autorisees[colonneSaisie] ou "nom"   // défaut sûr si valeur inconnue
  "... ORDER BY " + colonne                       // provient d'un ensemble fixe → sûr
  ```

- **Validation des entrées** — contraindre le type, le format, la plage des valeurs, en **liste blanche** (autoriser ce qui est connu) plutôt qu'en liste noire (interdire ce qui est connu comme mauvais, toujours contournable).
- **Moindre privilège** — le compte applicatif ne dispose que des droits nécessaires (chap. 10) ; ainsi, même en cas de faille, les dégâts sont **bornés** (pas de `DROP`, pas d'accès aux autres schémas). C'est une défense en profondeur, pas une parade primaire.
- **Échappement en dernier recours** — si l'on ne peut vraiment pas paramétrer (cas rare), utiliser la fonction d'échappement du pilote — mais c'est **fragile et déconseillé** : le paramétrage reste toujours préférable.
- **Procédures stockées** — sûres **uniquement** si elles paramètrent en interne ; une procédure qui construit du SQL dynamique par concaténation est **tout aussi vulnérable**.

---

## Anti-modèles fréquents

- **Concaténer / interpoler** une entrée dans le SQL — y compris via `%`/f-strings en Python (le piège du §17.1.2, où `%s` n'est **pas** un formatage de chaîne) ou via les *template literals* en JavaScript.
- **Croire qu'un ORM rend immunisé** — il protège par défaut, mais ses portes de sortie SQL ne le font pas.
- **Croire que l'échappement ou une liste noire suffit** — il faut paramétrer.
- **Construire un `ORDER BY` ou un identifiant** à partir d'une saisie brute, sans liste blanche.

---

## Spécificités et compléments MariaDB

- **Prepared statements** côté client ou serveur — le mécanisme et ses nuances sont au §17.9.
- **`sql_mode` n'est pas une défense anti-injection** — le mode strict valide les données (chap. 11.3), il ne protège pas contre l'injection.
- **MaxScale Database Firewall** (chap. 14.4.4) — peut bloquer des motifs d'injection au niveau réseau, en complément (jamais en remplacement) du paramétrage applicatif.
- **Audit et journalisation** (chap. 10.8) — pour **détecter** les tentatives a posteriori.

---

## Détecter et tester

La prévention se vérifie : l'**analyse statique** (SAST) et la **revue de code** repèrent les concaténations dangereuses ; les **tests de sécurité** (DAST) sondent l'application ; et la discipline de test (§17.6) couvre le comportement attendu. Traiter toute concaténation d'entrée dans le SQL comme un défaut bloquant est une bonne règle d'équipe.

---

## Ce qu'il faut retenir

- L'injection SQL vient de la **concaténation d'entrées non fiables** dans le SQL : on **mélange code et données**.
- La parade de référence est le **paramétrage systématique** (*prepared statements*, §17.9, §17.1.x), qui sépare le code des données — pas l'échappement.
- **Défense en profondeur** : ORM paramétré (attention aux portes SQL brutes), **liste blanche** pour les **identifiants** (non paramétrables), **validation** des entrées, **moindre privilège** (chap. 10).
- Les **procédures stockées** ne sont sûres que si elles paramètrent en interne.
- Compléments MariaDB : **MaxScale Firewall** (chap. 14.4.4) et **audit** (chap. 10.8) ; `sql_mode` n'est **pas** une défense anti-injection.
- **Vérifier** par SAST, revue de code et tests (§17.6) ; proscrire toute concaténation d'entrée dans le SQL.

⏭️ [Prepared statements et parameterized queries (curseurs sur prepared statements)](/17-integration-developpement/09-prepared-statements.md)

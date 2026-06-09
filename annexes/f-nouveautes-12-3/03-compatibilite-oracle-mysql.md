🔝 Retour au [Sommaire](/SOMMAIRE.md)

# F.3 — Compatibilité renforcée Oracle & MySQL

La série 12.x renforce sensiblement la compatibilité avec les deux principales sources de migration : **Oracle** — pour les organisations qui souhaitent réduire le coût de leurs licences — et **MySQL**, en particulier MySQL 8. Cette section regroupe par SGBD d'origine les nouveautés orientées compatibilité, jusque-là dispersées dans les chapitres. Elle complète le chapitre 19, qui traite la *démarche* de migration : [§19.1](../../19-migration-compatibilite/01-migration-depuis-mysql.md) pour MySQL, [§19.2.1](../../19-migration-compatibilite/02.1-depuis-oracle.md) pour Oracle.

## Compatibilité Oracle

Plusieurs de ces fonctionnalités s'inscrivent dans le **mode de compatibilité Oracle** (`SET sql_mode = ORACLE`), qui active une syntaxe et des comportements proches d'Oracle. Le SOMMAIRE signale explicitement la dépendance au mode pour certaines d'entre elles (jointures `( + )`, `DROP USER`) ; d'autres sont disponibles plus largement. Les chapitres référencés précisent, au cas par cas, le périmètre exact.

### Fonctions de conversion et de formatage

La série 12.x **complète** le jeu de fonctions de conversion emblématiques d'Oracle (§3.7.1) : `TO_DATE` (chaîne → date) en 12.3, `TO_NUMBER` (chaîne → nombre) et `TRUNC` (qui tronque une date à une unité, ou un nombre à un nombre de décimales donné) en 12.2. Elles rejoignent `TO_CHAR` (**date/heure** → texte, avec le modificateur `FM` qui supprime le remplissage), apparue plus tôt, dès la **10.6**. À noter : contrairement à Oracle, le `TO_CHAR` de MariaDB **ne convertit pas les nombres** en chaînes (un tel usage renvoie une erreur).

```sql
-- Fonctions de conversion façon Oracle
SELECT TO_DATE('09/06/2026', 'DD/MM/YYYY')   AS dt,       -- chaîne → DATE
       TO_NUMBER('1234.50')                  AS montant,  -- chaîne → nombre
       TO_CHAR(NOW(), 'FMDD/MM/YYYY')        AS libelle;  -- date → texte (FM : sans remplissage)
```

Le modèle de format exact (jetons reconnus, comportement de `FM`) est détaillé au §3.7.1.

### Jointures externes en syntaxe `( + )`

En mode Oracle, l'opérateur `( + )` offre une alternative à la syntaxe standard `LEFT` / `RIGHT JOIN` (§3.3.5). Il se place sur la colonne de la table « déficiente » — celle dont les lignes peuvent manquer.

```sql
-- Jointure externe en syntaxe Oracle ( + )
SELECT c.nom, o.montant
FROM clients c, commandes o
WHERE c.id = o.client_id (+);     -- équivaut à : clients c LEFT JOIN commandes o
```

Ici, tous les clients sont retournés, y compris ceux sans commande. Cette syntaxe facilite le portage de requêtes Oracle existantes, mais la forme standard `JOIN` reste recommandée pour le code neuf.

### Programmation procédurale façon PL/SQL

Trois apports rapprochent la programmation serveur de PL/SQL et facilitent le portage de packages Oracle. Les **tableaux associatifs** (`DECLARE TYPE … TABLE OF … INDEX BY`, §8.1.4) fournissent des collections en mémoire indexées par clé, utilisables au sein des routines. Le type **`SYS_REFCURSOR`** (un `REF CURSOR` faible, §8.5.2) permet de renvoyer ou de transmettre un jeu de résultats entre routines, la variable `max_open_cursors` bornant le nombre de curseurs ouverts simultanément. Enfin, les **curseurs sur *prepared statements*** (§8.5.1) autorisent un curseur à parcourir une requête préparée dynamiquement, un schéma courant dans le code Oracle.

### Type XML et résolution des routines

Le **type `XML` basique** (`XMLTYPE`, §2.2.6) facilite la migration de schémas Oracle reposant sur ce type ; il est volontairement *basique* et ne vise pas les fonctionnalités d'une base XML complète. La commande **`SET PATH`** (§19.2.1) définit un chemin de résolution des noms de routines non qualifiés : c'est en réalité un mécanisme du standard SQL, qui sert ici aussi bien aux migrations Oracle qu'à celles venant d'autres SGBD s'appuyant sur un chemin de recherche.

### Gestion des utilisateurs

Le comportement de `DROP USER` est aligné sur celui d'Oracle (§10.2.1) : la commande émet un avertissement lorsque l'utilisateur visé a des sessions actives, et **échoue en mode Oracle**. Comme il s'agit d'un changement de comportement, ce point est également repris en [F.4](04-impact-migration-compatibilite.md).

## Compatibilité MySQL

Côté MySQL, l'objectif est de lisser la migration depuis le serveur amont, notamment MySQL 8.

**Authentification (`caching_sha2_password`).** La 12.x prend en charge le plugin d'authentification par défaut de MySQL 8 (§10.5.5). Les applications et connecteurs configurés pour MySQL 8 peuvent ainsi s'authentifier sans reprovisionner les identifiants, ce qui simplifie aussi les environnements mixtes. Cette prise en charge vient en complément des plugins modernes propres à MariaDB, `ed25519` (§10.5.2) et PARSEC (§10.6).

**Fonctions géospatiales.** La 12.x s'aligne sur de nouvelles fonctions GIS introduites par MySQL 8 (§19.1.1), améliorant la portabilité des requêtes spatiales d'un serveur à l'autre.

## En synthèse

La compatibilité progresse sur deux fronts : réduire le coût d'un départ d'Oracle (fonctions de conversion, jointures `( + )`, constructions PL/SQL, type `XML`) et fluidifier la migration depuis MySQL 8 (authentification, fonctions géospatiales). Ces apports complètent les chapitres de migration, qui couvrent la démarche et les différences résiduelles à anticiper ([§19.1](../../19-migration-compatibilite/01-migration-depuis-mysql.md), [§19.2.1](../../19-migration-compatibilite/02.1-depuis-oracle.md), ainsi que §19.1.2 pour les points d'attention).

Le panorama complet des nouveautés figure en [F.1](01-tableau-recapitulatif.md) ; les changements de comportement — dont le cas de `DROP USER` en mode Oracle — sont analysés en [F.4](04-impact-migration-compatibilite.md).

⏭️ [Impact sur migration et compatibilité (changements de comportement)](/annexes/f-nouveautes-12-3/04-impact-migration-compatibilite.md)

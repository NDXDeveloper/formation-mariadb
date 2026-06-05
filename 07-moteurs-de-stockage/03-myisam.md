🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 7.3 MyISAM : Moteur legacy

> **Chapitre 7 — Moteurs de Stockage** · MariaDB 12.3 LTS

## Un moteur historique, devenu legacy

MyISAM est l'**ancêtre** des moteurs de stockage de l'écosystème MySQL/MariaDB. Dérivé du code ISAM, il a été le **moteur par défaut** de MySQL jusqu'à la version 5.5 (fin 2009), avant d'être supplanté par InnoDB. Sa réputation tenait à sa **simplicité** et à sa **rapidité** sur des charges en lecture, au prix d'absences majeures : ni transactions, ni clés étrangères, ni résistance aux pannes.

Aujourd'hui, MyISAM est un moteur **legacy** : on le rencontre encore dans des bases anciennes ou avec certains outils tiers, mais il **ne doit pas être choisi pour de nouveaux développements**. La documentation de MariaDB recommande explicitement de lui préférer le moteur **Aria** pour les nouvelles applications — Aria étant un successeur de MyISAM plus performant dans la plupart des cas et, surtout, conçu pour être *crash-safe* (§7.4). Pour tout besoin transactionnel, c'est InnoDB qu'il faut retenir (§7.2).

## Architecture et stockage

Une table MyISAM est stockée dans **trois fichiers** sur le disque, portant le nom de la table :

- `.frm` — la **définition** de la table (ce fichier existe pour tous les moteurs qui écrivent des données sur disque ; il relève du serveur, pas du moteur lui-même) ;
- `.MYD` — les **données** (*MY Data*) ;
- `.MYI` — les **index** (*MY Index*).

À noter que MariaDB **conserve les fichiers `.frm`**, là où MySQL 8 les a supprimés au profit d'un dictionnaire de données transactionnel : ne transposez donc pas les habitudes de l'un à l'autre. Les données étant écrites octet de poids faible en premier, les fichiers MyISAM sont **portables** d'une machine à l'autre par simple copie ; le fichier de données et le fichier d'index peuvent par ailleurs être placés sur des **disques distincts** pour gagner en performance.

Côté mémoire, MyISAM utilise un **cache de clés** (*key cache*), dimensionné par `key_buffer_size`, qui ne met en cache **que les blocs d'index** — la mise en cache des données proprement dites repose sur le cache de fichiers du système d'exploitation. C'est une différence fondamentale avec le buffer pool d'InnoDB (qui cache données *et* index). Le réglage du key cache est traité en §15.2.3.

```ini
[mariadb]
key_buffer_size = 128M   # cache d'index MyISAM ; les données reposent sur le cache de l'OS
```

## Caractéristiques et limites

Les forces et faiblesses de MyISAM découlent toutes de sa simplicité :

- **Pas de transactions** : aucun `COMMIT` ni `ROLLBACK`. Une instruction interrompue peut laisser les données dans un état partiel.
- **Pas de clés étrangères** : MyISAM accepte la syntaxe `FOREIGN KEY` mais l'**ignore** ; aucune intégrité référentielle n'est appliquée (voir §7.2.1).
- **Verrouillage au niveau table** : toute écriture verrouille la table entière, ce qui **sérialise les écrivains** et limite fortement la concurrence en écriture — à l'opposé du verrouillage au niveau ligne d'InnoDB.
- **Non *crash-safe*** : MyISAM n'a pas de journalisation pour rejouer ou annuler des modifications. En cas de crash ou de coupure de courant, les tables peuvent se **corrompre** et nécessiter une réparation.
- **Index FULLTEXT et spatiaux (GIS)** : MyISAM les prend en charge (un atout historique, désormais partagé par InnoDB).

Parmi ses limites de capacité figurent une taille maximale de 256 To par table, 64 index par table, 32 colonnes par index et une longueur d'index maximale de 1000 octets.

## Les rares atouts de MyISAM

Quelques caractéristiques peuvent encore jouer en sa faveur dans des cas très ciblés :

- **`SELECT COUNT(*)` instantané** : MyISAM conserve le nombre exact de lignes dans les métadonnées de la table ; un comptage sans clause `WHERE` est donc immédiat, là où InnoDB doit parcourir un index (conséquence du MVCC).
- **Empreinte compacte et fichiers simples**, faciles à copier ou à archiver.
- **Insertions concurrentes** : via `concurrent_insert`, MyISAM autorise l'ajout de lignes en fin de fichier pendant que des lectures ont lieu, tant qu'il n'y a pas de « trous » laissés par des suppressions.
- **Tables compressées en lecture seule** : l'outil `myisampack` produit des tables très compactes, adaptées à des données statiques.
- **Tables MERGE** : il est possible de construire une table MERGE au-dessus de plusieurs tables MyISAM de structure identique.

Ces avantages restent **étroits**. Surtout, Aria offre l'essentiel des bénéfices de MyISAM tout en étant *crash-safe*, ce qui rend les arguments en faveur de MyISAM rarement décisifs.

## Maintenance et réparation

Parce qu'il n'est pas *crash-safe*, MyISAM impose un **entretien réactif** que les moteurs modernes ont rendu inutile. En cas de corruption (requêtes qui s'interrompent, erreurs inattendues), on vérifie et répare les tables :

```sql
CHECK TABLE codes_pays;     -- diagnostiquer une éventuelle corruption
REPAIR TABLE codes_pays;    -- réparer la table
```

`REPAIR TABLE` fonctionne pour MyISAM (ainsi qu'Aria, Archive et CSV) ; l'outil hors-ligne équivalent est `myisamchk`. La vitesse de reconstruction d'un index dépend de `myisam_sort_buffer_size`, et l'option `USE_FRM` permet de recréer un index à partir du fichier `.frm` lorsque l'index est manquant ou son en-tête corrompu. Cette **charge de maintenance** est l'une des principales raisons de préférer un moteur résistant aux pannes.

## MyISAM dans MariaDB aujourd'hui

MyISAM n'est plus le moteur par défaut, et MariaDB n'en dépend pas pour son propre fonctionnement : le serveur s'appuie sur **Aria** (et non MyISAM) pour ses **tables temporaires internes** et la plupart de ses **tables système**. On peut donc créer une table MyISAM si nécessaire — par exemple une petite table de référence statique — mais c'est rarement le bon choix :

```sql
CREATE TABLE codes_pays (
  code CHAR(2)     PRIMARY KEY,
  nom  VARCHAR(80) NOT NULL
) ENGINE = MyISAM;
```

Dans la pratique, on rencontre encore MyISAM lors de **migrations de bases anciennes** ou avec des outils qui le présupposent. Le réflexe est alors de **convertir** les tables vers un moteur moderne :

```sql
ALTER TABLE ma_table ENGINE = InnoDB;   -- pour du transactionnel
-- ou
ALTER TABLE ma_table ENGINE = Aria;     -- équivalent crash-safe de MyISAM
```

La conversion entre moteurs et ses précautions sont détaillées en §7.9.

## Quand préférer un autre moteur

Le choix se résume simplement :

- besoin de **transactions, de clés étrangères ou de forte concurrence en écriture** → **InnoDB** (§7.2) ;
- envie d'un comportement **proche de MyISAM mais *crash-safe*** → **Aria** (§7.4).

La grille de décision complète entre tous les moteurs figure en §7.8.

## Liens avec d'autres chapitres

- **Aria**, le successeur recommandé de MyISAM, est présenté en §7.4.
- Le contraste avec **InnoDB** (transactions, clés étrangères, verrouillage au niveau ligne) est traité en §7.2 et §7.2.1.
- Le réglage du **key cache** MyISAM est abordé en §15.2.3.
- Les commandes **`CHECK TABLE` / `REPAIR TABLE`** sont détaillées en §11.6.
- La **comparaison des moteurs** (§7.8) et la **conversion entre moteurs** (§7.9) complètent le tableau.

## Ce qu'il faut retenir

- MyISAM est l'**ancien moteur par défaut**, aujourd'hui **legacy** : à connaître, mais à ne pas choisir pour de nouveaux projets.
- Il est **simple et rapide en lecture** mais **non transactionnel, sans clés étrangères, à verrouillage de table et non *crash-safe*** — donc exposé à la corruption en cas de panne.
- Une table MyISAM tient en **trois fichiers** (`.frm`, `.MYD`, `.MYI`) ; son cache (`key_buffer_size`) ne couvre que les **index**.
- Ses rares atouts (`COUNT(*)` instantané, compression `myisampack`, MERGE) sont **étroits**, et Aria les couvre en restant *crash-safe*.
- Pour du nouveau code, utilisez **InnoDB** (transactionnel) ou **Aria** (successeur *crash-safe* de MyISAM) ; convertissez les tables MyISAM héritées via `ALTER TABLE … ENGINE` (§7.9).

⏭️ [Aria : Le successeur de MyISAM](/07-moteurs-de-stockage/04-aria.md)

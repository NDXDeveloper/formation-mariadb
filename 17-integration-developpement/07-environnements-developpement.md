🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 17.7 Environnements de développement

Un bon environnement de développement est **cohérent, reproductible et proche de la production**. C'est ce qui évite le syndrome du « ça marche sur ma machine » : un bug qui n'apparaît qu'en production parce que l'environnement local en différait. Cette section traite de la mise en place de l'environnement MariaDB du développeur. Elle s'appuie sur la discipline de configuration (§17.4), l'installation de migrations (§17.5) et les tests (§17.6), et renvoie au chap. 16.3 (Docker), au chap. 1.9 (outils) et à l'annexe D.4 (configuration de développement local) pour les détails.

---

## Le principe de parité dev/prod

La méthodologie *Twelve-Factor App* recommande de garder **développement, *staging* et production aussi semblables que possible**. Pour la base de données, cela signifie concrètement :

- **développer contre MariaDB**, et non contre un substitut (SQLite, etc.) — c'est le pendant, côté développement, du principe vu pour les tests au §17.6 ;
- **utiliser la même version majeure** que la production (12.3) ;
- **rapprocher la configuration** là où elle influe sur le comportement (jeu de caractères `utf8mb4`, `sql_mode`, niveau d'isolation).

Chaque écart entre les environnements est une source potentielle de surprises.

---

## Où faire tourner MariaDB en local ?

Plusieurs options, avec leurs compromis :

- **Docker / Docker Compose** (recommandé) — MariaDB tourne dans un conteneur défini dans un fichier, identique pour toute l'équipe, à la version épinglée, isolé et jetable. Repose sur le chap. 16.3.
- **Installation native** — MariaDB installé directement sur le poste (chap. 1.8). Plus simple pour un usage isolé, mais moins reproductible et plus difficile à aligner sur la version des collègues.
- **Base de développement partagée** — une base centrale à laquelle tout le monde se connecte. Pratique, mais crée un **couplage** : les changements d'un développeur affectent les autres, et les données entrent en conflit.
- **Base cloud** — une instance managée par développeur ou partagée.

La recommandation usuelle : un MariaDB **par développeur**, en conteneur, reproductible et aligné en version.

---

## Docker Compose pour le développement

Un fichier `docker-compose.yml` décrit MariaDB (et souvent l'application, plus un outil d'inspection léger). À titre d'illustration :

```yaml
services:
  db:
    image: mariadb:12.3
    environment:
      MARIADB_DATABASE: boutique
      MARIADB_USER: app_user
      MARIADB_PASSWORD: ${DB_PASSWORD}          # lu depuis .env, non versionné
      MARIADB_ROOT_PASSWORD: ${DB_ROOT_PASSWORD}
    ports:
      - "3306:3306"
    volumes:
      - db-data:/var/lib/mysql                   # persistance des données

  adminer:                                        # interface web légère d'inspection
    image: adminer
    ports:
      - "8080:8080"

volumes:
  db-data:
```

La gestion fine des conteneurs, des volumes et de la persistance est traitée au chap. 16.3.2 et 16.3.3. À noter le compromis du **volume** : le conserver garde les données entre redémarrages ; le supprimer (`docker compose down -v`) repart d'une base vierge — utile pour retester l'enchaînement migrations + données.

---

## Configuration et secrets

C'est le fil rouge rappelé tout au long du chapitre (§17.1, §17.4), et l'environnement de développement en est le point d'ancrage. Conformément au *Twelve-Factor* (« la configuration vit dans l'environnement ») :

- **Variables d'environnement** pour les chaînes de connexion et identifiants — jamais codés en dur.
- **Fichier `.env` local, non versionné** (ajouté au `.gitignore`), accompagné d'un **`.env.example` versionné** servant de modèle sans valeurs réelles.
- **Une configuration distincte par environnement** (dev, test, *staging*, prod), via les variables d'environnement.
- **Un gestionnaire de secrets** (coffre-fort de type Vault, services cloud) pour les environnements supérieurs ; en local, le `.env` suffit généralement.

La règle absolue : **aucun secret réel dans le code ni dans le dépôt**.

---

## Mettre en place le schéma

Le schéma de l'environnement de développement se construit comme partout : en **appliquant les migrations** (§17.5), par le même mécanisme que sur les autres environnements. On évite les `ALTER TABLE` manuels ad hoc, qui réintroduiraient la dérive que les migrations sont censées éliminer.

---

## Données de développement (seed et anonymisation)

Pour travailler, un développeur a besoin de **données réalistes**. Trois approches :

- **Scripts de *seed*** — peupler une base neuve avec un jeu de données connu, versionné avec le code.
- **Données de production anonymisées** — il est **interdit d'utiliser des données de production brutes** contenant des informations personnelles en développement (confidentialité, conformité RGPD). Si l'on part de données réelles, elles doivent être **masquées/anonymisées** au préalable.
- **Données synthétiques** — générées artificiellement, sans donnée réelle.

Un bon *seed* est **minimal et focalisé**, juste assez pour développer et reproduire les cas utiles.

---

## Outils d'inspection

Pour explorer la base de développement, on dispose des outils vus au chap. 1.9 — le client en ligne de commande, **DBeaver**, **HeidiSQL**, **phpMyAdmin** — ou d'**Adminer**, particulièrement commode en conteneur (comme dans l'exemple Compose ci-dessus).

---

## L'échelle des environnements

Au-delà du poste de développement, on retrouve une **échelle** : développement → test → *staging* → production. Chacun doit être **isolé**, et le *staging* vise à **refléter la production** pour une validation finale. La configuration diffère d'un environnement à l'autre uniquement par les variables d'environnement, le code et les migrations restant identiques.

---

## Reproductibilité et accueil des nouveaux

L'objectif ultime : qu'un **nouveau développeur** obtienne un environnement fonctionnel en quelques commandes — typiquement *démarrer les conteneurs*, *appliquer les migrations*, *charger le seed*. On passe ainsi du « ça marche sur ma machine » à « ça marche dans le conteneur, à l'identique pour tout le monde ». La configuration MariaDB de développement peut d'ailleurs être plus **légère** que celle de production (*buffer pool* réduit, etc.) — voir l'annexe D.4.

---

## Ce qu'il faut retenir

- Viser la **parité dev/prod** : développer contre **MariaDB** (même version majeure, 12.3), pas un substitut (§17.6).
- Privilégier un MariaDB **par développeur en conteneur** (Docker Compose, chap. 16.3), reproductible et jetable ; attention au compromis de **persistance des volumes**.
- **Configuration dans l'environnement** : variables d'environnement, **`.env` non versionné** + `.env.example`, config par environnement, **aucun secret dans le dépôt** (§17.4).
- Construire le schéma par **migrations** (§17.5), pas par changements manuels.
- Fournir des **données de développement** par *seed*, **anonymisation** (jamais de données de prod brutes) ou génération synthétique.
- Maintenir une **échelle d'environnements** isolés (dev/test/staging/prod), un *staging* fidèle à la prod, et un **accueil rapide** des nouveaux (chap. 1.9 pour les outils, annexe D.4 pour la config).

⏭️ [Prévention des injections SQL](/17-integration-developpement/08-prevention-injections-sql.md)

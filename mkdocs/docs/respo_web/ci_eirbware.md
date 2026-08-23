# CI/CD Eirbware

Afin de faciliter la maintenance ainsi que le déploiement des services proposés par Eirbware ainsi que les sites internets des clubs/assos, nous avons mis en place des méthodes de déploiement automatique.

Ces méthodes reposent sur des github actions, qui permettent à chaque push/merge dans la branche principale d'un repo d'exécuter des commandes.

Celles dont nous nous servont sont définies dans [ce dépôt](https://github.com/Eirbware/.github/), comme des actions réutilisables (dans le dossier `.github/workflows`).

Ainsi, pour utiliser une de nos actions, on peut définir un job ayant cette forme:
```yaml
  build-and-push:
    permissions:
      id-token: write
    uses: Eirbware/.github/.github/workflows/build-and-publish.yml@master
    with:
      file: "./Dockerfile"
      image_name: ${{ github.repository }}
    secrets:
      registry_username: ${{ secrets.registry_username }}
      registry_api_key: ${{ secrets.registry_api_key }}
```

Il y 2 types d'entrées à ces jobs:

- Les paramètres, sous l'option `with`
- Les secrets, sous l'option `secrets`, qui contient les données sensibles dont le job a besoin

!!!warning
    Il est vivement conseillé d'utiliser les jobs de cette façon pour déployer sur notre serveur, ainsi si un changement survient il n'y aura pas de modification à faire sur les dépôts utilsant l'action.

!!!info
    Pour définir les secrets et les utiliser comme vu au dessus, il faut aller dans les options du dépôt, dans secrets and variables et dans actions

!!!info
    Pour les services étant sous l'organisation github d'Eirbware, on peut aussi utiliser `secrets: inherit`

## Les services statiques

Pour les sites statiques, nous mettons à disposition un job permettant de copier les fichiers du site vers le serveur.
Ce job a cette forme:
```yaml
deploy-static:
    uses: Eirbware/.github/.github/workflows/deploy-static.yml@master
    with:
        remote_user: "www-eirb"
        directory_to_copy: "./dist"
    secrets:
        ssh_secret_key: ${{ secrets.ssh_private_key }}
        ssh_public_key: ${{ secrets.ssh_public_key }}
        ssh_cert: ${{ secrets.ssh_cert }}
```

Ce job va copier les fichiers présents dans le dossier `directory_to_copy` (ici `dist`) dans le dossier `www-<user>/nginx/www` via sftp.

Les paramètres de ce job sont:

* `remote_user` (obligatoire): Le nom de l'utilisateur sur nos serveurs (usuellement `www-<nom-du-service>`)
* `directory_to_copy` (obligatoire): Le dossier source, contenant les fichiers du site
* `artifact_name` (ignoré par défaut): Nom de l'artéfact à utiliser (voir [ici](https://docs.github.com/en/actions/concepts/workflows-and-actions/workflow-artifacts) pour la documentation sur le sujet)
* `artifact_path` (ignoré par défaut, obligatoire si `artifact_name` est non nul): Chemin vers lequel l'artéfact doit être copié

## Les services conteneurisées

Le déploiement des services conteneurisées se fait en 2 parties: le build/publish de l'image et le déploiement de celle-ci.

### Build/Push

Le job permettant de faire cela a cette forme:
```yaml
  build-and-push:
    permissions:
      id-token: write
    uses: Eirbware/.github/.github/workflows/build-and-publish.yml@master
    with:
      file: "./Dockerfile"
      image_name: ${{ github.repository }}
    secrets:
      registry_username: ${{ secrets.registry_username }}
      registry_api_key: ${{ secrets.registry_api_key }}
```

!!!warning
    Il ne faut pas omettre l'option `permissions: id-token: write`, sans quoi l'action ne fonctionnera pas.

Il va compiler le docker puis publier l'image sur [registry.eirb.fr](https://registry.eirb.fr), d'où il pourra être pull par sur le serveur.

Les paramètres de ce job sont:

* `file` (obligatoire): L'emplacement du `Dockerfile` dans le dépôt
* `image_name` (obligatoire): Le nom donné à l'image, il est conseillé de mettre `${{ github.repository }}`
* `context` (`.` par défaut): Le contexte dans lequel le build se déroule
* `artifact_name` (ignoré par défaut): Le nom de l'artéfact donné au job (voir [ici](https://docs.github.com/en/actions/concepts/workflows-and-actions/workflow-artifacts) pour la documentation sur le sujet)
* `artifact_path` (ignoré par défaut, obligatoire si `artifact_name` est non nul): Le chemin de l'artéfact donné au job

Ensuite, ce job a besoin de 2 secrets: `registry_username` et `registry_api_key`.

Si vous n'avez pas le `registry_username`, contactez-nous pour créer un compte, ce secret sera l'email associé au compte (plus de détails [ici](../gestion_vps/registry.md) pour le côté administrateur).

Une fois que cela est fait, connectez vous avec ce compte sur [le registry](https://registry.eirb.fr), puis en haut à gauche accédez à l'écran `API keys`.
A partir de là, vous pouvez créer une clé api, que vous devrez mettre dans le secret `registry_api_key`.

!!!info
    Nous vous conseillons de créer une clé api par service, afin de pouvoir éventuellement supprimer/renouveler une clé d'un service sans affecter les autres.

!!!warning
    Lors de la création de la clé, une date d'expiration doit être fixée, n'oubliez pas de renouveler la clé si la date est proche !

### Déploiement

Afin de déployer le service sur notre serveur, il faut maintenant automatiquement pull l'image du registry après l'avoir build et push.

#### Docker compose

Dans le dépôt du service, il faut d'abord créer un fichier docker compose qui sera celui utilisé par le serveur.
Ainsi dans cette configuration aucune image ne doit être build, car elles devront toutes être pull depuis une registry public ou notre registry.

Un exemple de ce genre de configuration est le [wiki](https://github.com/Eirbware/wiki.eirb.fr/):
```yaml
## MediaWiki with MariaDB (https://hub.docker.com/_/mediawiki)
##
## Access via "http://localhost:8080"
##   (or "http://$(docker-machine ip):8080" if using docker-machine)
services:
  mediawiki:
    image: registry.eirb.fr/eirbware/wiki.eirb.fr:main
    container_name: mediawiki
    restart: unless-stopped
    ports:
      - ${WEB_PORT:-8080}:80
    depends_on:
      - wiki-mariadb
    environment:
      # - USE_DEBUG=true
      - SITE_NAME=wiki.eirb.fr
      - SERVER=${SERVER}
      - MYSQL_DBSERVER=wiki-mariadb
      - MYSQL_DATABASE=wiki
      - MYSQL_USER=wiki
      - MYSQL_PASSWORD=${MYSQL_PASSWORD}
      - KC_PROVIDER=https://connect.eirb.fr/realms/eirb
      - KC_CLIENT_ID=wiki
      - KC_CLIENT_SECRET=${KC_CLIENT_SECRET}
      - WG_SECRET_KEY=${WG_SECRET_KEY}
      - WG_UPGRADE_KEY=${WG_UPGRADE_KEY}
    volumes:
      - ${PATH_TO_FILES}/wiki/images:/var/www/html/images

  # https://docs.linuxserver.io/images/docker-mariadb
  wiki-mariadb:
    image: lscr.io/linuxserver/mariadb:latest
    container_name: wiki-mariadb
    environment:
      - PUID=0
      - PGID=0
      - TZ=Etc/UTC
      - MYSQL_ROOT_PASSWORD=${MYSQL_ROOT_PASSWORD}
      - MYSQL_DATABASE=wiki
      - MYSQL_USER=wiki
      - MYSQL_PASSWORD=${MYSQL_PASSWORD}
    volumes:
      - ${PATH_TO_FILES}/mariadb:/config
    restart: unless-stopped
```

#### Portainer

Afin de gérer les différents environments, nous utilisons [portainer](https://www.portainer.io/), donc pour pouvoir déployer un service conteneurisé sur notre infrastructure, il faut d'abord que nous vous donnions accès à un environment.

Ensuite, dans l'environment, il faut aller dans Stacks->Add stacks.
Une fois dans cet écran, donnez un nom à votre stack (nous utilisons www-\<service\> par exemple) et sélétionnez l'option `Repository`.
Après cela, remplissez les informations nécessaires (à minima Repository URL)

!!!info
    Si votre dépôt est privé (pas très compréhensible mais pourquoi pas), il faudra aussi mettre les logins d'un compte ayant un accès en lecture au dépôt. Evitez donc de mettre un compte github personnel, mais créez plutôt un compte à l'association, afin d'assurer une meilleure passation.

Ensuite, il faut activer les `GitOps updates` et choisir le méchanisme `Webhook`.
Gardez le lien, il servira pour l'étape de fin.

Enfin, il ne reste plus qu'à ajouter les variables d'environment et à mettre l'access control sur `Restricted` en authorisant la team du même nom que l'environment.

Une fois que cela est fait, si l'image n'a pas encore été publiée, le conteneur ne pourra pas être lancé. Veillez donc à exécuter l'action définie précédemment avant de procéder.

#### Dire à portainer de mettre à jour le service

Enfin, rajouter un job au workflow ayant cette forme:
```yaml
deploy:
    needs: [ build-and-push ]
    uses: Eirbware/.github/.github/workflows/deploy-stack.yml@master
    # portainer_stack_webhook must be defined
    secrets:
        portainer_stack_webhook: ${{ secrets.portainer_stack_webhook }}
        wireguard_conf: ${{ secrets.wireguard_conf }}
```

La seule chose dont il y a besoin, c'est de 2 secrets:

* `portainer_stack_webhook` (obligatoire): c'est le lien récupéré lors de l'étape précédente
* `wireguard_conf` (obligatoire): configuration vpn pour pouvoir communiquer avec portainer, nous vous la donnerons

!!!warning
    Ne pas oublier d'utiliser le champ `needs: [ <job-before-1>, <job-before-2>, ... ]`, sans quoi portainer tentera de déployer la nouvelle version du service avant que l'image ait fini d'être publiée

!!!info
    Pour les services dont les dépôts sont sur l'organisation github d'Eirbware, il n'y a pas besoin de configurer la variable `wireguard_conf`, car elle est déjà configurée pour toute l'organisation.

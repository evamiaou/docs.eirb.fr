# CI/CD manuel

Cette page explique comment créer sa propre pipeline de déploiement d'un site statique, mis en exemple avec un site construit dans un projet npm et déployé par une pipeline [Gitlab](https://gitlab.com/). L'objectif est de garder un protocole aussi simple que possible afin de pouvoir l'adapter à des technologies différentes facilement soi-même ou avec l'aide d'un⋅e membre d'Eirbware. Si vous utilisez github ou que vous déployez un site utilisant un conteneur, vérifiez d'abord si une option de [CI Eirbware](./ci_eirbware.md) convient déjà.

!!!warning
    Cette page considère que vous disposez déjà d'un accès FTP manuel vers votre emplacement dans le serveur Eirbware. Si ce n'est pas le cas, commencez par [cette page](./acces_site.md).

## Le fichier de CI

Ceci est un exemple de fichier de CI pour gitlab (`.gitlab-ci.yml`) à mettre à la racine du projet. Les valeurs à remplacer sont:

- `MONASSO`, à remplacer par le nom de votre asso/club. Par exemple, pour `essaim.eirb.fr`, on mettra `essaim`.
- `VERSION_NODE` à remplacer par la version de node à utiliser (faire `node --version`), par exemple `22`, suivi éventuellement d'une version de debian pour fixer le comportement de `apt`, par exemple `22-trixie`. Vous pouvez consulter les tags disponibles [ici](https://hub.docker.com/_/node/tags)
- Éventuellement `dist` à la dernière ligne si le site construit ne se situe pas dans le répertoire `dist/`.

!!!info
    Si vous n'utilisez pas node, vous pouvez simplement utiliser une [image debian](https://hub.docker.com/_/debian), par exemple `bookworm`, ou une autre image de votre choix correspondant à la technologie que vous utilisez.

```yml title=".gitlab-ci.yml"
publish:
    image: "node:VERSION_NODE"
    stage: deploy
    only:
        - main
    script: |
        set -v
        #
        # === 1: Install lftp ===
        #
        apt update
        apt install lftp
        #
        # === 2: Configure ssh ===
        #
        mkdir -p ${HOME}/.ssh
        echo "$SSH_PRIV_KEY" | base64 --decode > ${HOME}/.ssh/id_MONASSO
        echo "$SSH_CERT" | base64 --decode > ${HOME}/.ssh/id_MONASSO-cert.pub
        echo "$SSH_CONFIG" | base64 --decode > ${HOME}/.ssh/config
        echo "$SSH_KNOWN_HOSTS" | base64 --decode > ${HOME}/.ssh/known_hosts
        chmod 400 ${HOME}/.ssh/*
        #
        # === 3: Build ===
        #
        npm install
        npm run build
        #
        # === 4: Deploy ===
        #
        lftp -c "open -u www-MONASSO, sftp://MONASSO; mirror -eRv dist nginx/www"
```

!!!warning
    Si vous utilisez une plateforme autre que Gitlab, ce fichier aura un nom différent et une structure différente. Seul le script restera similaire.

## Explication du fichier CI

`image: "node:VERSION_NODE"`: Cela spécifie quelle image utiliser pour le conteneur qui va éventuellement construire votre site, puis l'envoyer vers le serveur eirbware. Dans ce cas, on utilise une image node car cela permet d'avoir npm pré-installé dans une version contrôlée. On pourrait aussi utiliser une image debian et installer une autre technologie manuellement.

`only: - main`: Cela signifie à Gitlab que cette pipeline ne doit être exécutée que lors d'un commit sur la branche `main`. Cela vous permet de créer une autre branche de travail qui ne sera pas déployée.

`script`: Cette balise indique le début du script qui sera exécuté sur le conteneur. Le conteneur dispose d'une copie de tout le code dans votre dépôt à la version actuelle et le script est exécuté à la racine de cette copie.

## Explication du script

`set -v` permet de copier toutes les commandes exécutées dans la sortie console, afin de monitorer et débugger le déroulement de votre pipeline, qui ne fonctionnera pas du premier coup.

La première vraie étape consiste à installer le logiciel `lftp`. lftp est un outil permettant de cloner un répertoire ftp sans interaction, ce qui sera fait à la fin du script. Cette commande fonctionne sur les images basées sur debian trixie. À priori, elle devrait fonctionner sur toute image debian pendant de nombreuses années, mais il se pourrait qu'il faille un jour changer d'outil ou changer de commande d'installation.

L'étape 2 configure les fichiers SSH. Les fichiers sont stockés dans des variables qu'on définira dans la partie [Variables SSH](#variables-ssh) ci-dessous. Ils sont décodés en base64 car certains fichiers ont besoin de retours à la ligne, qui ne peuvent pas être stockés dans des variables CI gitlab.

L'étape 3 consiste à construire le site web. Cette étape dépend complètement des technologies utilisées. Dans le cas d'un site vanilla, cette étape n'est pas nécessaire et peut être supprimée.

L'étape 4 copie le site depuis le conteneur vers le serveur Eirbware. lftp est utilisé avec l'option `-c` pour lui passer sans interaction les commandes à exécuter. D'abord la commande de connexion, puis la commande `mirror`:

- `-eRv`
    - `e` pour supprimer les fichiers distants qui n'existent pas dans la nouvelle version.
    - `R` pour copier les fichier du conteneur vers le serveur distant (au lieu de l'inverse).
    - `v` pour augmenter la verbosité de l'opération, voir chaque fichier copié/supprimé à distance.
- `dist`: Le nom du dossier local où le site construit se trouve. Par exemple, à partir de la racine de votre projet, `dist/index.html` correspond à la page racine du site.
- `nginx/www`: Le nom du dossier distant dans le serveur eirbware. Cette option ne devrait à priori pas être modifiée.

## Variables SSH

### Générer les variables

Pour cette étape, nous conseillons de créer un nouveau répertoire sur votre machine, dans lequel on va créer 4 fichiers.

Pour fonctionner, votre pipeline aura besoin des 4 variables `SSH_PRIV_KEY`, `SSH_CERT`, `SSH_CONFIG`, et `SSH_KNOWN_HOSTS`. Ces 4 variables correspondent respectivement aux 4 fichiers qui seront créés sur le conteneur: `~/.ssh/id_MONASSO`, `~/.ssh/id_MONASSO-cert.pub`, `~/.ssh/config`, et `~/.ssh/known_hosts`.

Vous disposez normalement déjà d'une clé privée et d'un certificat pour accéder au serveur (sinon, lire [cette page](./acces_site.md)). Par la suite, nous appellerons ces fichiers `id_MONASSO` et `id_MONASSO-cert.pub`. Copiez ces 2 fichiers dans le répertoire créé.

La variable `SSH_CONFIG` sera créée à partir d'un fichier qu'on va appeler `config` dans notre répertoire, ayant cette forme:

```title="config"
Host MONASSO
    port 30
    Hostname eirb.fr
    User www-MONASSO
    IdentityFile    ~/.ssh/id_MONASSO
    CertificateFile ~/.ssh/id_MONASSO-cert.pub
```

Enfin, la variable `SSH_KNOWN_HOSTS` sera créée à partir d'un fichier qu'on va appeler `known_hosts` dans notre répertoire. La manière la plus simple de créer ce fichier est d'effectuer une première connexion depuis son poste en enregistrant le serveur eirbware comme hôte connu si la question est posée, puis, après avoir fermé la connexion, effectuer cette commande qui va copier la signature du serveur dans un nouveau fichier: `grep eirb.fr ~/.ssh/known_hosts > known_hosts`. Le fichier créé devrait ressembler à cela:

```
[eirb.fr]:30 ssh-ed25519 <signature>
```

Si ces 4 étapes se sont déroulées normalement, votre repertoire devrait ressembler à cela:

```
.
├── config
├── id_MONASSO
├── id_MONASSO-cert.pub
└── known_hosts
```

Pour obtenir les valeurs des variables à fournir à Gitlab, exécutez la commande `base64 <fichier> -w 0` sur chacun de ces fichiers. Cette commande va coder les fichiers en [base64](https://fr.wikipedia.org/wiki/Base64) pour les contenir dans une chaîne sans retour à la ligne, et les fichiers seront décodés dans le script exécuté par le conteneur.

Par exemple, on exécute `base64 config -w 0` pour obtenir la valeur de la variable `SSH_CONFIG` qu'on donnera à Gitlab.

### Enregistrer les variables dans Gitlab

!!!warning
    Cette étape est écrite pour Gitlab. Si vous utilisez une plateforme différente, cette partie n'est pas applicable.

!!!warning
    Il arrive que l'interface de Gitlab change et que des choses soient renommées. Si c'est le cas, demandez de l'aide à un membre d'Eirbware pour vous aider à trouver où se trouvent les options et pour signaler qu'il faut mettre cette documentation à jour.

Dans la page d'accueil de votre projet, allez dans "Settings"/"Paramètres", puis "CI/CD". Ensuite, dans "Variables", cliquez sur "Add variable"/"Ajouter une variable". Pour chacune des 4 variables qu'on va ajouter, il va falloir paramétrer 6 éléments:

- Visibilité: Pour les variables `SSH_CONFIG` et `SSH_KNOWN_HOSTS`, on peut se permettre de laisser le paramètre par défaut. Pour les 2 autres variables, il faut sélectionner "Masked and hidden"/"Masqué et caché", car il s'agit de secrets et elle ne doivent pas être révélées.
- "Protect variable"/"Protéger la variable": Dans l'idéal, activer pour les 4 variables. Cela empêche que la variable ne soit lue et fuitée par un commit sur une nouvelle branche par un contributeur malveillant (seules les personnes peuvent contribuer, mais c'est par sécurité). Cela nécessitera de s'assurer que la branche utilisée pour le déploiement est activée comme branche protégée. Si cela vous paraît compliqué, désactivez le paramètre.
- "Expand variable reference": À désactiver
- Description: Laisser vide
- "Key"/"Clé": Le nom de la variable (par exemple `SSH_CONFIG`)
- "Value"/"Valeur": La valeur donnée après le codage en base64 à l'étape précédente.

## Conclusion

Normalement, votre pipeline de déploiement devrait désormais se lancer lors du prochain commit sur la branche `main` avec le fichier `.gitlab-ci.yml`. Pour suivre le déroulement de l'exécution, après avoir poussé le commit, rendez-vous sur la page du projet. Vous aurez, en haut de l'interface, un résumé du dernier commit, et une icône en cours / succès / échec qui vous mènera au suivi de l'exécution.

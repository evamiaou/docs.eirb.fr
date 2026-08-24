## Gestion des utilisateurs

Cette section se concentre sur :

* La création d'un nouvel administrateur
* L'archivage d'un administrateur
* L'ajout d'un accès à un respo web
* La révocation d'un accès `SSH`/`SFTP`

### Création d'un nouvel administrateur

La création d'un administrateur se fait facilement à l'aide des scripts
d'Eirbware, vous devrez vous occuper de ces 3 étapes :

* Créer l'utilisateur
* Créer un accès `SSH`
* Créer un accès wireguard

#### Création de l'utilisateur administrateur

Afin de créer un l'utilisateur pour un nouvel administrateur, vous devez
utiliser le script `new_admin` en tant qu'administrateur comme suivant :

```title="Création d'un nouvel administrateur"
sudo new_admin <cas_login>  # Créé l'utilisateur adm-<cas_login>
```

!!!info "Changement du mot de passe"

    Ce script créé un mot de passe temporaire pour l'administrateur, il devra
    **mettre à jour** son mot de passe lors de sa première connexion par `SSH`.

#### Création de l'accès `SSH`

Comme dit précédemment, l'accès par `SSH` se fait par clé et certificat, le
nouvel administrateur devra créer une paire de clés spécifiquement pour
Eirbware en utilisant la commande suivante :

```title="Création d'une paire de clés `SSH` par le futur administrateur"
ssh-keygen -t ed25519
```

Il devra ensuite transmettre la clé publique à l'utilisateur qui créé l'accès
`SSH` afin de créer le certificat.

!!!info "Stockage des clés publiques"

    Les clés publiques des utilisateurs sont stockées dans le dossier
    `/int/int-ssh/public_keys` avec le nom `id_<cas_login>.pub`, cela permet de
    créer un nouveau certificat sans avoir à retransmettre la clé publique.

    Il peut être utile de créer un nouveau certificat si un admin veut utiliser
    un accès `SFTP` ou si un respo web s'occuper de plusieurs sites

Finalement, la création du certificat pour l'administrateur se fait à l'aide de
la commande suivante :

```title="Création d'un certificat `SSH` pour un administrateur"
sudo cert_new -k /int/int-ssh/public_keys/id_<cas_login>.pub adm-<cas_login>
```

!!!info "Passphrase du certificat d'Eirbware"

    Afin de signer la clé publique, la passphrase du certificat d'Eirbware est
    nécessaire, vous la trouverez sur le [Vaultwarden d'Eirbware](https://vault.eirb.fr).

#### Création de l'accès wireguard

L'accès au `VPN` se fait comme pour n'importe quel utilisateur, référez-vous
directement à [cette section de la documentation](vpn.md#wireguard).

### Archivage d'un administrateur

En vrai c'est pas dispo atm

### Ajout d'un accès à un respo web

Donner les accès au respo web d'un asso à son site se fait en 2 étapes:

* Créer un accès `SSH`
* Créer un accès wireguard

#### Création de l'accès ssh utilisateur

Similairement à la création d'un administrateur, le respo web devra créer une paire de clés spécifiquement pour Eirbware, de la manière suivante:

```title="Création d'une paire de clés `SSH` par le futur administrateur"
ssh-keygen -t ed25519
```

Il devra ensuite transmettre sa clé publique à un administrateur d'Eirbware.

!!!info "Stockage des clés publiques"

    Comme dit précédemment, les clés publiques des utilisateurs sont stockées dans le dossier
    `/int/int-ssh/public_keys` avec le nom `id_<cas_login>.pub`.

Ensuite, celui-ci créera un certificat avec la commande suivante:

```title="Création d'un certificat `SSH` pour un administrateur"
sudo cert_new -k /int/int-ssh/public_keys/id_<cas_login>.pub www-<nom_site> <cas_login>
```

!!!info "Passphrase du certificat d'Eirbware"

    Afin de signer la clé publique, la passphrase du certificat d'Eirbware est
    nécessaire, vous la trouverez sur le [Vaultwarden d'Eirbware](https://vault.eirb.fr).

#### Création de l'accès wireguard

Comme pour la création d'un administrateur, référez-vous à [cette page](vpn.md#wireguard)

### Révocation d'un accès `SSH`/`SFTP`

Afin de révoquer l'accès à un utilisateur, il faudra d'abord obtenir l'identifiant de son certificat, via la commande `cert_list`.

Une fois ceci fait, la commande est:

```title="Révocation d'un certificat ssh"
sudo cert_revoke <certificate_ID>
```

!!!info "Réactiver un certificat"

    Si un certificat a été révoqué par erreur, il est possible d'annuler cette action avec l'option `-r` de la commande
    `cert_revoke`

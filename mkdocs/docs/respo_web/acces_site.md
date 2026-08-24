# Comment demander un accès pour mettre à jour un site

La modification d'un site web se fait par `SFTP`, qui est un protocole construit
à partir de `SSH`. Et l'accès se fait exclusivement par clé.

Les instructions suivantes permettent de savoir comment créer une clé `SSH` et
la configuration pour mettre à jour un site.

## Création d'une clé `SSH`

Il est conseillé de créer une clé `SSH` spécifiquement pour Eirbware, vous
pouvez en créer une de la façon suivante :

```title="Création d'un clé `SSH`"
ssh-keygen -t ed25519
```

Après l'exécution de cette commande, deux fichiers ont été créés : une clé
privée `filename` et une clé publique `filename.pub`.

!!!warning "Sécurité des clés"

    Vous **ne devez pas, sous aucun prétexte, partager la clé privée**, cela
    permettrait à un tiers d'usurper votre identité.

    Cependant, vous pouvez partager la clé publique, elle ne sert à rien sans la
    clé privée.

## Certificat `SSH`

Eirbware utilise des [certificats `SSH`](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/6/html/deployment_guide/sec-using_openssh_certificate_authentication)
afin d'autoriser la connexion d'une paire de clés `SSH` au serveur. Pour en
créer un, Eirbware a besoin de la clé publique que vous venez de générer pour
créer un certificat.

Vous devez donc à partir de là contacter un membre d'Eirbware comme proposé en
haut de cette page pour demander de vous créer un certificat.

!!!info "Durée de validité des certificats"

    Par défaut, les certificats **sont valides 1 an**, vous devrez peut-être
    demander un renouvellement de votre certificat.

## Configuration `SSH`

Finalement, vous pouvez créer une configuration `SSH`, vous devez rajouter dans
votre fichier `~/.ssh/config` les lignes suivantes (exemple pour pixeirb, vous
devez les adapter au site que vous maintenez) :

```title="~/.ssh/config" linenums="1"
Host eirb_pix
    port 30
    Hostname eirb.fr
    User www-pix
    IdentityFile    ~/.ssh/<Votre clé SSH>
    CertificateFile ~/.ssh/<Votre certificat SSH>
```

Avec ceci, vous avez tout pour passer à l'étape d'après : Comment mettre à jour
un site ?

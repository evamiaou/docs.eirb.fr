# Eirbconnect

Afin de faciliter l'authentification par cas pour les services hébergés par Eirbware, Eirbconnect a été créé.
Il permet d'offrir un point d'entrée plus simple utilisant OpenId, un protocole supporté nativement par énormément de librairies.

Pour cela nous utilisons [keycloak](https://www.keycloak.org/), qui permet à la fois de gérer une authentification en interne et de communiquer avec des services d'authentification extérieurs.

La documentation utilisateur se trouve [ici](../respo_web/connexion_cas.md).

## Accéder à la console admin

Afin d'accéder à la console admin, il faut obligatoirement passer par [vpn.eirb.fr](https://vpn.eirb.fr), et se connecter en utilisant le compte `eirbware` (les codes sont dans le vaultwarden).

Aller directement à l'adresse [https://connect.vpn.eirb.fr](https://connect.vpn.eirb.fr) ne permettra pas de se connecter (car il faut passer par Eirbconnect).

## Gestion des utilisateurs

En allant dans le panel `Users`, il est possible de voir la liste de tous les utilisateurs possédant un compte Eirbconnect, à partir de la il est possible de faire plusieurs choses, notamment:

* Ajouter un utilisateur, avec le bouton Add user. Il faudra au minimum renseigner username et email, mais renseigner first name et last name est conseillé
* Supprimer un utilisateur
* Réinitialiser le mot de passe d'un utilisateur. Pour cela, il faut aller dans le panel `Credentials`, puis `Reset Password`

## Gestion des clients

Un client est un service qui utilise l'authentification (par exemple bde pour [bde.eirb.fr](https://bde.eirb.fr)).
Il est déconseillé de créer manuellement un client, car le script de création de site (cf [ici](services.md)) le fait automatiquement.
Leur gestion se fait via le panel `Clients`.

### Récupérer le secret d'un client

Le secret d'un client se trouve dans le panel `Credentials` du client.
Il est aussi possible de le regénérer via le bouton `Regenerate`.

### Url de redirection valides

En plus d'avoir besoin du secret du client, il faut que l'url vers laquelle l'authentification redirige soit valide.
La liste des urls valide se trouve dans l'option `Valid redirect URIs` dans le panel `Settings`.

Par défaut, les urls acceptés sont:

* `https://<service>.eirb.fr/*`
* `http://localhost:8080/*`

### Authentification directe par CAS

Pour certains services, notamment ceux qui utilisent l'école ou l'année, passer par Eirbconnect risque de renvoyer des données qui ne sont pas à jour.

Dans ces cas là, il est judicieux de forcer la connection par cas.
Pour cela, il faut aller dans le panel `Advanced` du client, et tout en bas, dans la section `Authentication flow overrides` changer le champ `Browser Flow` en sélectionnant `idp redirector`.

## Identity provider (Idp)

Afin de pouvoir communiquer avec CAS, Keycloak passe par le panel `Identity provider`.

Actuellement, deux providers sont définis:

* CAS, qui communique par le cas via un module custom.
* saml-shibboleth, qui utilise le module saml par défaut de keycloak pour communiquer avec le service [shibboleth](https://www.shibboleth.net) de la dsi qui lui communiquera avec CAS.

Actuellement, les attributs transmis par la DSI sont les suivants:

* email (courriel)
* firstName (prénom)
* lastName (nom)
* ecole (ecole)
* diplome (diplôme)
* username (celui-ci est particulier, il se sert du username template importer, qui filtre la partie correspondant à l'identifiant cas de la personne)

!!!info
    Si d'autres attributs sont nécessaires, il faut contacter la DSI via un ticket pour leur faire la demande.

Le module CAS provoquant quelques problèmes (par exemple, lors de la création d'un compte non élève), seul le module saml-shibboleth est utilisé aujourd'hui

!!!info
    Les users ont d'autres champs, mais ceux-ci peuvent rester vides, ce sont des reliquats de l'utilisation du module CAS

La traduction des attributs se fait via des mappers, définis dans le panel `Mappers`.

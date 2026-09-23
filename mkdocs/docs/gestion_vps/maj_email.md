# Mise à jour d'email

Lorsqu'un⋅e étudiant⋅e change de prénom d'usage, son adresse mail sera mise à jour. Au-delà des mises à jour à faire dans l'interface web du keycloak pour l'email et le prénom, il est nécessaire d'effectuer une manipulation complexe pour permettre la connexion via le cas dur Eirbconnect.

## Connexion

D'abord la manipulation nécessite de modifier directement la base de données de Eirbconnect. Nous allons nous connecter en CLI.

D'abord, connectez-vous au serveur. Ensuite, connectez-vous à l'utilisateur connect: `sudo su www-connect -`.

Ensuite, se connecter au conteneur en mode interactif: `docker exec -it connect-postgres bash`.

Enfin, se connecter à la base de données avec psql: `psql --username=connect-admin --dbname=connect-db`.

## Effectuer là mise à jour

Pour obtenir l'UUID de l'utilisateur, exécuter cette requête dans psql:

```sql
SELECT user_id FROM federated_identity WHERE federated_user_id='{cas}'; -- par exemple federated_user_id='alovelace'
```

Ensuite, mettre à jour les données:

```sql
UPDATE federated_identity
SET
    federated_user_id='{Prenom}.{Nom}@bordeaux-inp.fr', -- prénom et nom avec majuscule
    federated_username='{prenom}.{nom}@bordeaux-inp.fr' -- sans majuscule
WHERE
    user_id='{uuid}' AND -- l'UUID trouvé à l'étape précédente
    identity_provider='saml-shibboleth'
;
```

Normalement, à partir de cette étape, la connexion à Eirbconnect via le CAS devrait fonctionner.

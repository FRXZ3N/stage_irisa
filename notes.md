# Prise de notes Keycloak/OpenID Connect

Open Source -> Faille de sécurité ?

## WebAuthn&SSO

>https://webauthn.guide

WebAuthn (Web Authentification API) permets d'utiliser les données biométriques pour l'authentification des utilisateurs, tels que le scanner d'empreintes digitales ou la reconnaissance faciale. En effet, cela évite d'oublier des mots de passes trop complexes, ou d'éviter les fuites ou cracks des mots de passes utilisateurs.

### WebAuthnAPI

Permet aux serveurs d'utiliser les authentificateurs présents dans les appareils, tels que Windows Hello ou Touch ID d'Apple. Au lieu du mot de passe, une paire clé publique/clé privée (**Credentials**) va être crée pour un site
- Clé privée : Stockée de manière sécurisée sur l'appareil de l'utilisateur 
- Clé publique : Liée à un credential ID généré aléatoirement, stocké sur le serveur. Le serveur va utiliser la clé publique pour prouver l'identité de l'utilisateur\
Non secrète car inutile sans la clé privée correspondante

Trois propriétés importantes :
- Strong : Backed by Hardware Security Module, stockage sécurisé des clés privées et execution des operations cryptographiques dont a besoin WebAuthn
- Scoped : La paire de clés est utile seulement pour une origine spécifique (Cookies par exemple). De ce fait, une paire de clés enregistrée à "webauthn.guide" ne pourra pas être utilisée à "evil-webauthn.guide", limitant les risques de phishing
- Attested : Les autorités de certification peuvent fournir un certificat qui aide les serveurs à vérifier que la clé publique provient bien d'un authentificator trustable et non d'une source frauduleuse.

**1. Navigator.credentials.create()**

Création d'un nouveau Credential (Object publicKeyCredentialCreationOptions) avec différents paramètres :
- challenge : Buffer d'octets générés aléatoirement et cryptographié sur le serveur, utlise pour prévenir des "replay attacks" (interception du hash password pour retransmission)
- rp (relying party) : Description de l'organisation responsable de l'enregistrement et de l'authentification de l'utilisateur
    - name
    - id -> Doit être un sous-ensemble du domaine actuellement sur le navigateur
- user : Informations sur l'utilisateur
    - id -> Utilisé pour associer le credential et l'utilisateur
    > ⚠️ Pas d'informations personnelles dans l'id
    - name -> Nom d'utilisateur (ici, une adresse mail)
    - displayName -> Nom affiché
- pubKeyCredParams : Tableau d'objets décrivant les clés publiques acceptables par le serveurs
    - alg : Nombre décrit par le registre COSE
    > https://www.iana.org/assignments/cose/cose.xhtml#algorithms
    - type
- authenticatorSelectors (Optionnel) : Aide la rp à mettre plus de restrictions pour l'enregistrement
    - authenticatorAttachment : Indication sur la méthode d'authentification (platform -> Windows Hello/ Apple's Touch ID, cross-platform(Yubikey))
- timeout : Temps pour que l'utilisateur réponde avant une erreur
- attestation : Permet de gérer l'importance pour l'enregistrement de la donnée "attestation" (Contient des informations pour tracer l'utilisateur)
    - None : Pas d'importance pour le serveur
    - Indirect : Données d'attestations anonyme
    - Direct : Le serveur veut les données d'attestation venant de l'authenticator

L'objet retourné par la fonction contient la clé publique ainsi que d'autres informations:
- id : id du credential généré. Il est utilisé pour identifié le credential lors de l'authentification de l'utilisateur.
- rawId : Id, mais en binaire
- AuthenticatorAttestationResponse :
    - clientDataJSON : Représente les données passées du navigateur à l'authentificator, afin d'associer le nouveau credential avec le serveur et le navigateur.
    - attestationObject : Contient la clé publique, un certificat d'attestation optionne, ainsi que d'autres métadonnées
- type (ici, public-key)

**2. Parsing and validating the Registration Data**
> Validation en 19 points

Conversion du fichier tableau d'octets en JSON

**3. navigator.credentials.get**

Obtention d'un credential, avec sa signature (Object publicKeyCredentialCreationOptions):
- challenge : Same as create()
- allowCredentials : Ce tableau dit au navigateur avec quels credentials le serveur veut que l'utilisateur s'identifie:
    - id : credentialID
    - type (ici, public-key)
    - transports (Optionnel) : protocole de transport (ex: USB, BT, NFC)
- timeout : Same as create()

L'objet retourné est un PublicKeyCredential. Il contient des informations un peu différentes que l'objet de l'enregistrement :
- id : L'identifier utilisé pour générer l'assertion d'authentification
- rawId: id en binaire
- AuthenticatorAssertionResponse :
    - authenticatorData : Similaire à attestationObject, mais ne contient pas la clé publique. Utilisé pour générer la signature.
    - clientDataJSON : Same as registration
    - signature : Signature générée par l'association de la clé privée et du credential. Sur le serveur, la clé publique sera utilisée pour verifier que cette signature est valide
    - userHandle (Optionnel) : Représente l'id fourni durant l'enregistrement. Peut être utilisé pour confirmer l'assertion de l'utilisateur sur le serveur

**4. Parsing and Validating the Authentication Data**

Après obtention de l'assertion, elle est envoyé au serveur pour validation. La signature est ensuite validée grâce aux clés publiques stockées dans la base de données du serveur.


### Red Hat's SSO : Realm configuration

Les realms sont utilisés pour gérer des ensembles d'utilisateurs, credentials, roles et groupes. Les realms sont isolés les uns des autres et peuvent gérer uniquement les utilisateurs qu'ils contrôlent.\
Afin d'utiliser WebAuthn dans Keycloak, il faut configurer WebAuthnRegister comme une action par défaut lors d'une connexion :
- WebAuthn Browser flow : Execution du WebAuthn Authenticator dans le flux du navigateur
    - Username Password Form
    - WebAuthn Authenticator
- WebAuthn Registration flow : Execution du WebAuthn Authenticator dans le flux d'enregistrement
    - Registation User Creation
    - Profile Validation
    - Password Validation
    - Recaptcha

> Attention: il faut bien bind le Browser Flow et le Registration Flow à la nouvelle configuration avec WebAuthn

### Red Hat's SSO : Client for biometric authentication

Les clients sont des entités qui nécessite l'utilisation d'un SSO pour authentifier un utilisateur. Le plus souvent, les clients sont des applications ou des services qui veulent utliser SSO pour se sécuriser eux-mêmes et proposer une solution de connexion. Ils peuvent également demander les informations d'identité ou un token d'accès pour accéder à d'autres services de manière sécurisée.\

- Installation :
    - Spécification du realm
    - Url de redirection pour l'authentification
    - SSL
    - Resource
    - Booléen pour serveur public ou non
    - Port de confidentialité

### Exemple de déploiement

- React
>https://developers.redhat.com/articles/2021/11/09/biometric-authentication-webauthn-and-sso#deploy_a_react_client_to_test_webauthn

- Angular
>https://github.com/mauriciovigolo/keycloak-angular

## Otka

Otka n'étant pas Open Source, il faut envisager des alternatives :

- **Ory**
>https://www.ory.sh

- Auth0
>https://auth0.com

- Azure Active Directory
>https://azure.microsoft.com/en-us/services/active-directory/

- AWS Cognito
>https://aws.amazon.com/cognito/

## Spring Security

>https://github.com/spring-projects/spring-security
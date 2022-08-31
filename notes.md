# Prise de notes Keycloak/OpenID Connect

Open Source -> Faille de sécurité ?

# Table of contents

- [WebAuthn&SSO](#webauthnsso)
  - [WebAuthnAPI](#webauthnapi)
  - [SSO](#sso)
  - [Red Hat's SSO Client](#red-hats-sso--client-for-biometric-authentication)
  - [Red Hat's realm config](#red-hats-sso--realm-configuration)
- [Okta](#okta)
- [Spring Security](#spring-security)
- [JWT](#jwt)

## WebAuthn&SSO

> https://webauthn.guide

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

### SSO

`Single Sign-On : Méthode d'authentification qui permet aux utilisateurs d'accéder à plusieurs services ou applications avec un seul jeu d'identifiants`

> https://www.okta.com/fr/blog/2021/02/single-sign-on-sso/

1. Fonctionnement

Basée sur le concept de l'identité fédérée (partage des attributs d'identité sur des systèmes fiables et autonomes), c'est à dire que lorsque qu'un utilisateur est approuvé par un système, il reçoit automatiquement les droits d'accès à tous les autres systèmes ayant établi une relation de confiance avec le premier système. OpenIDConnect et SAML 2.0 sont des protocoles de bases des solutions SSO avancées.\
Lors de la connexion d'un utilisateur à un service passant par une connexion SSO, un jeton d'authentification est crée et stocké dans le navigateur ou dans les serveurs de la solution SSO. De ce fait, chaque site web ou application effectuera une vérification auprès du service SSO, qui enverra ce jeton pour confirmer l'identité de l'utilisateur et lui fournir l'accès.

2. Types de SSO

- **Security Access Markup Language (SAML)** : Norme ouverte qui encode du texte en langage machine et échange les informations d'utilisation. Cette norme est utilisée pour aider les fournisseurs d'applications à vérifier que les demandes d'authentification sont appropriées. Optimisée pour une utilisation dans les applications web, autorisant la transmission des informations via un navigateur

- **Open Authorization (OAuth)** : Protocole d'autorisation basé sur une norme ouverte qui transfert les informations d'identification entre application, et les chiffre en code machine. Les utilisateurs peuvent ainsi autoriser une application à accéder à leurs données dans une autre application sans avoir besoin de valider manuellement leur identité. Utile pour les applications natives.

- **OpenID Connect (OIDC)** : Se place au dessus d'OAuth 2.0 pour ajouter des informations sur l'utilisateur et autoriser le processus d'authentification SSO. Ce protocole permet d'accéder à plusieurs applications avec une seule session de connexion. Exemple : Connexion à une application via un compte Google ou Facebook.

- **Kerberos**: Protocole permettant une authentification mutuelle, dans laquelle l'utilisateur vérifie l'identité du serveur, et inversement, le tout sur des connexions réseau non sécurisées. Il utilise un service d'attribution de tickets qui emet des jetons pour authentifier les utilisateurs et les applications.

- **Authentification par carte à puce** : Utilisation d'équipements matériels (ex: carte à puce que l'utilisateur connecte à son ordinateur), pour réaliser le processus de SSO. Le logiciel interagit avec les clés cryptographiques de la carte pour authentifier l'utilisateur. Bien qu'elles soient autement sécurisée, l'utilisateur est le risque (matériel portable donc risque de perte) et l'utilisation est coûteuse (Yubikey : de 29 à 90 € pour un utilisateur, de 650 à 950 € pour un serveur)

3. Avantages

- **Surface d'attaque réduite** : la SSO élimune la manque de rigueur sur les mots de passe, donc limite le phishing. Il n'y a besoin que d'un seul mot de passe fort

- **Accès utilisateurs fluides et sécurisés** : la SSO fournit des renseignements en temps réél sur les utilisateurs, ainsi que sur le moement et l'emplacement de la connexion, permettant aux entreprises de protéger l'intégrité de leurs systèmes. Limitations des risques de sécurité : l'équipe IT peut désactiver l'accès d'un terminal aux comptes et données critiques de l'entreprise.

- **Audit simplifié des accès utilisateurs** : la SSO peut servir à configurer les droits d'accès utilisateurs en fonction de leur rôle ou niveau hierarchique. Assurance de la transparence et de la visibilité sur les niveaux d'accès.

- **Utilisateurs plus autonomes et productifs** : la SSO permet de supprimer le besoin de suivi manuel, autorisant un accès immédiat en un seul clic à des milliers d'applications. Fin de la gestion manuelle des demandes.

- **Pérennisation** : Première étape de la protection d'une entreprise.

4. Défis

- **Risques liés aux accès utilisateurs** : Réussir à obtenir des identifiants SSO fournit l'accès à toutes les applications auxquelles l'utilisateurs à accès. Il est donc nécessaire de déployer d'autres mécanismes d'authentification

- **Vulnérabilités potentielles** : Des vulnérabilités dans les protocoles SAML et OAuth permettent un accès non autorisé. Il faut donc associer la SSO avec d'autres facteurs d'authentification et de gouvernance des identités

- **Compatibilité des applications** : Toutes les applications ne s'intègrent pas efficacement dans une solution SSO. Il faut donc dans la conception de l'application, prévoir une vraie fonctionnalité SSO, auquel cas, la couverture ne sera pas complète.

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

- Test avec téléphone : prise en charge de la lecture biométrique et enregistrement de l'appareil.

### Exemple de déploiement

- React

  > https://developers.redhat.com/articles/2021/11/09/biometric-authentication-webauthn-and-sso#deploy_a_react_client_to_test_webauthn

- Angular
  > https://github.com/mauriciovigolo/keycloak-angular

## Okta

Okta n'étant pas Open Source, il faut envisager des alternatives :

- **Ory**

  > https://www.ory.sh >https://github.com/ory

- Auth0

  > https://auth0.com

- Azure Active Directory

  > https://azure.microsoft.com/en-us/services/active-directory/

- AWS Cognito

  > https://aws.amazon.com/cognito/

- **OpenStack Keystone**
  > https://docs.openstack.org/yoga

## Spring Security

> https://github.com/spring-projects/spring-security

## JWT

`JSON Web Token`

Internet standard for creating data with signature and encryption. Tokens are signed either using a private secret or a public/private key.

- Structure :
  - Header :
    Indentifies which algorithm is used to generate the signature
  ```
  {
      "alg": ... // Cryptographic algorithm, here HMAC_SHA256
      "typ": "JWT"
  }
  ```
  - Payload :
    Contains a set of claims
  ```
  {
      "loggedInAs": "admin" // Custom claim
      "iat": 1422779638 // Issued At Time claim
  }
  ```
  - Signature :
    Securely validates the token. Calculated by encoding the header and payload and concatening the two together with a period separator. That string is run through the cryptographic algorithm specified in the header
  ```
  HMAC_SHA256(
      secret,
      base64urlEncoding(header) + '.' +
      base64urlEncoding(payload)
  )
  ```

Finally,\
 `const token = base64urlEncoding(header) + '.' + base64urlEncoding(payload) + '.' + base64urlEncoding(signature)`

- Use :
  When a user logs in using credentials, a JWT is returned and saved locally. Clients may also authenticate directly by generating and signing its own JWT and pass it to a OAuth service :

```
POST /oauth2/token?
Content-type: application/x-www-form-urlencoded

grant_type=urn:ietf:params:oauth:grant-type:jwt-bearer&assertion=eyJhb...
```

If the client passes a valid JWT assertion, the server generate an access_token

```
{
    "access_token": "eyJhb..."
    "token_type": "Bearer"
    "expires_in": 3600
}
```

- Standards fields : Standard claim fields & Commonly-used header fields

- Implementations : Nearly every langages

- Vulnerabilities :
  - JWT could become stateless, undermining its primary advantage
  - Breachs on `alg` field

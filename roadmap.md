# RoadMap pour le plugin d'authentification adaptative au sein de Keycloak

## Utiliser WebAuthn pour envoyer les informations à Keycloak depuis React

![keycloak](_resources/keycloak.png)\
Sur cette capture d'écran est affiché le fichier `keycloak.json`. Il permet de spécifier le realm auquel appartient l'utilisateur, et permettre de le rediriger vers celui-ci. En effet, le realm est indiqué dans l'URL après la connexion, juste après `http://localhost:8080/auth` qui est la redirection de base.

![clientConfig](_resources/clientConfig.png)\
On voit que `"resource": "app"`, la capture d'écran ci-dessus indiquant la configuration de ce type d'utilisateur. On peut voir que cette configuration autorise la redirection depuis `http://localhost:3000/*`, qui est l'adresse d'hébergement de l'application React, ainsi que toute les pages découlantes de cette URL.

![](_resources/realmConfigGen.png)\
La configuration du realm `Demo` (indiqué sur le `keycloak.json`). On y indique seulement le nom, si il doit être activé, et ses endpoints (Possibilités de récupérer les fichiers).

![](_resources/realmConfigLog.png)\
La configuration du login du realm permet à un nouvel utilisateur de s'enregistrer (Possibilité de desactivation si on veut une création de compte utilisateur uniquement par l'administrateur), ainsi qu'un certain nombre de fonctionnalités, comme l'oubli de mot de passe, demander au navigateur de se souvenir de l'utilisateur, ou encore d'utiliser son email comme nom d'utilisateur.

![](_resources/authConfigBrowserFlow.png)\
Ici, copie de la browser base authentication, pour y ajouter l'exexcution de l'authenticator WebAuthn dans les requis.

![](_resources/authConfigRegistrationFlow.png)\
ici, copie du registration flow de base, pour y ajouter WebAuthn Authenticator (Possibilité de ne pas le mettre en `required`).

Utilisation d'une nouvelle instance de l'objet `Keycloak`, prenant en paramètres le fichier de configuration [keycloak.json](#utiliser-webauthn-pour-envoyer-les-informations-à-keycloak-depuis-react)

## Ajouter plus d'informations à celles de base

### Ordinateur

Ecrire un script dans le développement du plugin pour lister les périphériques disponibles :

- Linux : `lsusb`
- Windows : `Get-PnPDevice`
- MacOS : `system_profiler SPUSBDataType`
- Mobiles ?
  Afin d'établir l'intersection entre contexte environnemental et contexte "hardware" (Problème de deployabilité)

### Capteur Téléphone

- Gyroscope (Téléphone en mouvement, changement de démarche)
- Luxmètre (Eclairement de la pièce)
- Caméra (Reconnaissance faciale)
- Lecteur biométrique (Reconnaissance de l'empreinte digitale)
- Microphone (Reconnaissance vocale)

Etablissement d'un chemin d'authentification selon les capteurs présents

### Exemple FranceConnect

> https://partenaires.franceconnect.gouv.fr/fcp/fournisseur-service#identite-pivot

## Analyser l'API REST afin d'ajouter des flows de manière programmique

- Authentication Management

  > https://www.keycloak.org/docs-api/18.0/rest-api/index.html#_authentication_management_resource

  - Add new authentication execution

  ```
  POST /{realm}/authentication/executions
  ```

  - Update execution with new configuration

  ```
  POST /{realm}/authentication/executions/{executionId}/config
  ```

  - Lower/Raise execution's priority

  ```
  POST /{realm}/authentication/executions/{executionId}/lower-priority
  POST /{realm}/authentication/executions/{executionId}/raise-priority
  ```

  - Create a new authentication flow

  ```
  POST /{realm}/authentication/flows
  ```

  - Add new authentication execution to a flow

  ```
  POST /{realm}/authentication/flows/{flowAlias}/executions/execution
  ```

### Analyse du code openSource

- CredentialModel (DEPRECATED)

  - id
  - type
  - userLabel
  - createdDate
  - secretData
  - credentialData

- OTPCredentialModel

  - String secretValue
  - String subType
  - int digits
  - int counter
  - int period
  - String algorithm

    - credentialData (subType, digits, counter, period, algorithm)
    - secretData (secretValue)

  - createTOTP()
    > Time One Time Password
  - createHOTP()
    > HMAC One Time Password
  - createFromCredentialModel()
  - fillCredentialModelFields()

## Adaptation dynamique, application de stratégie

Export du fichier .jar pour intégration au sein de Keycloak (via le folder standalone/deployments)\

Choix de la classe abstraite à _extends_ ?

- AbstractIdentityProviderMapper (exemple de github ssh key mapper)
- AbstractAuthenticationExecutionRepresentation
- AbstractAuthenticationFlowContext

Ajout des informations de context hardware dans le plugin ?
Architecture globales, déploiements ?

**Deploiement du JAR du plugin dans /standalone/deployments !**

## Analyse du code de l'extension IBM

### Dépendances

- Keycloak :
  - keycloak-core
  - keycloak-server-spi-private
  - keycloak-server-spi
- Autres :
  - httpclient
  - jboss-jaxrs-api_2.1_spec
  - jboss-logging
  - resteasy-jaxrs

### META-INF/services

- com.ibm.security.verify.authenticator.otp.IBMSecurityVerifyOtpLoginAuthenticatorFactory
- com.ibm.security.verify.authenticator.webauthn.fido2.IBMSecurityVerifyFido2LoginAuthenticatorFactory
- com.ibm.security.verify.authenticator.webauthn.registration.IBMSecurityVerifyFidoRegistrationRequiredActionAuthenticatorFactory
- com.ibm.security.verify.authenticator.verify.push.IBMSecurityVerifyPushNotificationLoginAuthenticatorFactory
- com.ibm.security.verify.authenticator.verify.qr.IBMSecurityVerifyQrLoginAuthenticatorFactory
- com.ibm.security.verify.authenticator.verify.registration.IBMSecurityVerifyRegistrationRequiredActionAuthenticatorFactory

### theme-resources (???)

- help !

### Main

- demo

  - `IBMSecurityVerifyDemoLoginAuthenticator`

    ```
    public void action(AuthenticationFlowContext context)
    ```

    > Activation du logger\
    > Recupération des paramètres de la requête HTTP\
    > Activation du bon mécanisme d'authentification en fonction du paramètre récupéré\
    > Fin du logger

    ```
    public void authenticate(AuthenticationFlowContext context)
    ```

    > Activation du logger\
    > Verification de l'activation de FIDO ou QRCode\
    > Initialisation des flows QR et FIDO\
    > Envoi des deux flows au navigateur pour l'utilisateur\
    > Rendu d'un bouton pour username/password qui bypass les options

    ```
    public void close()
    ```

    > Fermeture du logger

    ```
    public boolean configuredFor(KeycloakSession session, RealmModel realm, UserModel user)
    ```

    > Ouverture du logger\
    > Retourne true\
    > Fermeture du logger

    ```
    public boolean requiresUser()
    ```

    > Ouverture du logger\
    > Set requiresUser à "False"\
    > Fermeture du logger

    ```
    public void setRequiredActions(KeycloakSession session, RealmModel realm, UserModel user)
    ```

    > Unused

  - `IBMSecurityVerifyDemoLoginAuthenticatorFactory implements Authenticator Factory`

    > Configuration de "property"

    ```
    public void close()
    ```

    > Ouverture et fermeture du logger d'erreur

    ```
    public Authenticator create(KeycloakSession session)
    ```

    > Return IBMSecurityVerifyDemoLoginAuthenticator object

    ```
    Getters des paramètres
    ```

    ```
    public void init(Scope config)
    ```

    > Ouverture/Fermeture du logger "init"

    ```
    public boolean isConfigurable()
    ```

    > return True

    ```
    public boolean isUserSetupAllowed()
    ```

    > return false

    ```
    public void postInit(KeycloakSessionFactory factory)
    ```

    > Ouverture/Fermeture du logger "postnit"

- otp (ONE TIME PASSWORD)

  - `IBMSecurityVerifyOtpLoginAuthenticator`

    > Constantes pour les templates

    ```
    public void action(AuthenticationFlowContext context)
    ```

    > Récuperation des requêtes HTTP et code en fonction du paramètre récupéré\
    > Email ou SMS

    ```
    public void authenticate(AuthenticationFlowContext contextt)
    ```

    > Configuration de l'OTP en fonction du phone number ou email

  - `IBMSecurityVerifyOtpLoginAuthenticatorFactory implements AuthenticatorFactory`

    > Même que DemoLoginAuthenticationFactory

- rest
  - `FidoUtilities`
  - `FormUtilities`
  - `IBMSecurityVerifyUtilities`
  - `OtpUtilities`
  - `PushNotificationUtilities`
  - `QrUtilities`
  - `VerifyAppUtilities`
- utils

  - `IBMSecurityVerifyLoggingUtilities`

  > Gestion des erreurs avec un Logger

- verify

  - push (Push notifications authentication)

    - `IBMSecurityVerifyPushNotificationLoginAuthenticator`

      ```
      public void action(AuthenticationFlowContext context)
      ```

      > Récupération des paramètres de la requête HTTP\
      > Récupération de l'utilisateur\
      > Vérification de l'authentification, et erreur si mauvaise

      ```
      public void authenticate(AuthenticationFlowContext context)
      ```

      > Appel de initiatePushNotification(context)

      ```
      public void initiatePushNotification(AuthenticationFlowContext context)
      ```

      > Envoi de la notification push si l'utilisateur est non null, sinon création et redirection vers la page d'erreur

      ```
      public void requireVerifyRegistration(AuthenticationFlowContext context, String methodName)
      ```

      > Ajout de l'erreur avec message

    - `IBMSecurityVerifyPushNotificationLoginAuthenticatorFactory`

      > Getters pour les paramètres du constructeur de `property`

  - qr (QR Code authentication)
    - `IBMSecurityVerifyQrLoginAuthenticator`
    - `IBMSecurityVerifyQrLoginAuthenticatorFactory`
  - registration (Registration authentication)
    - `IBMSecurityVerifyRegistrationRequiredActionAuthenticator`
    - `IBMSecurityVerifyRegistrationRequiredActionAuthenticatorFactory`

- webauthn
  - fido2 (FIDO2 authentication [Linked with WebAuthn])
    - `IBMSecurityVerifyFido2LoginAuthenticator`
    - `IBMSecurityVerifyFido2LoginAuthenticatorFactory`
  - registration
    - `IBMSecurityVerifyFidoRegistrationRequiredActionAuthenticator`
    - `IBMSecurityVerifyFidoRegistrationRequiredActionAuthenticatorFactory`
- `AbstractIBMSecurityVerifyAuthneticatorFactory`

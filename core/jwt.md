# JWT Authentification

> [JSON Web Token (JWT)](https://jwt.io/) is a JSON-based open standard ([RFC 7519](https://tools.ietf.org/html/rfc7519)) for creating access tokens that assert some number of claims. For example, a server could generate a token that has the claim "logged in as admin" and provide that to a client. The client could then use that token to prove that he/she is logged in as admin. The tokens are signed by the server's key, so the server is able to verify that the token is legitimate. The tokens are designed to be compact, URL-safe and usable especially in web browser single sign-on (SSO) context.

[Wikipedia](https://en.wikipedia.org/wiki/JSON_Web_Token)

API Platform allows to easily add a JWT-based authentication to your API using [LexikJWTAuthenticationBundle](https://github.com/lexik/LexikJWTAuthenticationBundle).

API Platform is fully working with [LexikJWTAuthenticationBundle](https://github.com/lexik/LexikJWTAuthenticationBundle).

In order to install [the bundle please follow their documentation](https://github.com/lexik/LexikJWTAuthenticationBundle/blob/master/Resources/doc/index.md).

`LexikJWTAuthenticationBundle` requires your application to have a properly configured user provider. You can either use [API Platform's FOSUserBundle integration](fosuser-bundle) or  [create a custom user provider](http://symfony.com/doc/current/security/custom_provider.html).

Here's a sample configuration using the data provider provided by FOSUser:

```yml
# app/config/security.yml

security:
    encoders:
        FOS\UserBundle\Model\UserInterface: bcrypt

    role_hierarchy:
        ROLE_READER: ROLE_USER
        ROLE_ADMIN: ROLE_READER

    providers:
        fos_userbundle:
            id: fos_user.user_provider.username
            
    firewalls:
        login:
            pattern:  ^/login
            stateless: true
            anonymous: true
            provider: fos_userbundle
            form_login:
                check_path: /login_check
                username_parameter: email
                password_parameter: password
                success_handler: lexik_jwt_authentication.handler.authentication_success
                failure_handler: lexik_jwt_authentication.handler.authentication_failure
                require_previous_session: false

        main:
            pattern:   ^/
            provider: fos_userbundle
            stateless: true
            anonymous: true
            lexik_jwt: ~

        dev:
            pattern:  ^/(_(profiler|wdt)|css|images|js)/
            security: false

    access_control:
        - { path: ^/login, role: IS_AUTHENTICATED_ANONYMOUSLY }
        - { path: ^/books, roles: [ ROLE_READER ] }
        - { path: ^/, roles: [ ROLE_READER ] }
```       

## Testing with Behat
You can test your application with Behat like described in the doc and LexikJWTAuthenticationBundle by adding to `features/bootstrap/FeatureContext.php` these functions:
```php
    // features/bootstrap/FeatureContext.php
    
    // createDatabase and dropDatabase functions (note: the order is important)
    /**
     * @BeforeScenario @login
     *
     * use the https://symfony.com/doc/current/security/entity_provider.html#creating-your-first-user hash
     */
    public function login(\Behat\Behat\Hook\Scope\BeforeScenarioScope $scope) {
        $user = new \AppBundle\Entity\User();
        $user->setUsername('admin');
        $user->setPassword('$2a$08$jHZj/wJfcVKlIwr5AvR78euJxYK7Ku5kURNhNx.7.CSIJ3Pq6LEPC');
        $user->setEmail('test@test.com');
        $user->setNom('Test');

        $this->manager->persist($user);
        $this->manager->flush();

        $token = $this->jwtManager->create($user);

        $this->restContext = $scope->getEnvironment()->getContext('Behatch\Context\RestContext');
        $this->restContext->iAddHeaderEqualTo('Authorization', 'Bearer ' . $token);
    }

    /**
     * @AfterScenario @logout
     */
    public function logout() {
        $this->restContext->iAddHeaderEqualTo('Authorization', '');
    }
```

The `jwtManager` variable is just the `lexik_jwt_authentication.jwt_manager` service added to `behat.yml`.
Then, you just have to add to your features **@login** and **@logout** like **@createSchema** and **@dropSchema**.


Previous chapter: [FOSUserBundle Integration](fosuser-bundle.md)

Next chapter: [NelmioApiDocBundle integration](nelmio-api-doc.md)

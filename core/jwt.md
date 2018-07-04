# JWT Authentication

> [JSON Web Token (JWT)](https://jwt.io/) is a JSON-based open standard ([RFC 7519](https://tools.ietf.org/html/rfc7519)) for creating access tokens that assert some number of claims. For example, a server could generate a token that has the claim "logged in as admin" and provide that to a client. The client could then use that token to prove that he/she is logged in as admin. The tokens are signed by the server's key, so the server is able to verify that the token is legitimate. The tokens are designed to be compact, URL-safe and usable especially in web browser single sign-on (SSO) context.

[Wikipedia](https://en.wikipedia.org/wiki/JSON_Web_Token)

API Platform allows to easily add a JWT-based authentication to your API using [LexikJWTAuthenticationBundle](https://github.com/lexik/LexikJWTAuthenticationBundle).
To install this bundle, [just follow its documentation](https://github.com/lexik/LexikJWTAuthenticationBundle/blob/master/Resources/doc/index.md).

## Installing LexikJWTAuthenticationBundle

`LexikJWTAuthenticationBundle` requires your application to have a properly configured user provider.
You can either use the [Doctrine user provider](https://symfony.com/doc/current/security/entity_provider.html) provided
by Symfony (recommended), [create a custom user provider](http://symfony.com/doc/current/security/custom_provider.html)
or use [API Platform's FOSUserBundle integration](fosuser-bundle.md).

Here's a sample configuration using the data provider provided by FOSUserBundle:

```yaml
# app/config/packages/security.yaml
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
            json_login:
                check_path: /login_check
                username_path: email
                password_path: password
                success_handler: lexik_jwt_authentication.handler.authentication_success
                failure_handler: lexik_jwt_authentication.handler.authentication_failure

        main:
            pattern:   ^/
            provider: fos_userbundle
            stateless: true
            anonymous: true
            guard:
                authenticators:
                    - lexik_jwt_authentication.jwt_token_authenticator

        dev:
            pattern:  ^/(_(profiler|wdt)|css|images|js)/
            security: false

    access_control:
        - { path: ^/login, role: IS_AUTHENTICATED_ANONYMOUSLY }
        - { path: ^/books, roles: [ ROLE_READER ] }
        - { path: ^/, roles: [ ROLE_READER ] }
```

## Documenting the Authentication Mechanism with Swagger/Open API

Want to test the routes of your JWT-authentication-protected API?

### Configuring API Platform

```yaml
# api/config/packages/api_platform.yaml
api_platform:
    swagger:
         api_keys:
             apiKey:
                name: Authorization
                type: header
```

And the "Authorize" button will automatically appear in Swagger UI.

![Screenshot of API Platform with Authorize button](images/JWTAuthorizeButton.png)

### Adding a New API Key

All you have to do is configuring the API key in the `value` field.
By default, [only the authorization header mode is enabled](https://github.com/lexik/LexikJWTAuthenticationBundle/blob/master/Resources/doc/index.md#2-use-the-token) in [LexikJWTAuthenticationBundle](https://github.com/lexik/LexikJWTAuthenticationBundle).
You must set the [JWT token](https://github.com/lexik/LexikJWTAuthenticationBundle/blob/master/Resources/doc/index.md#1-obtain-the-token) as below and click on the "Authorize" button.

```
Bearer MY_NEW_TOKEN
```

![Screenshot of API Platform with the configuration API Key](images/JWTConfigureApiKey.png)


## Testing with Behat

Let's configure Behat to automatically send an `Authorization` HTTP header containing a valid JWT token when a scenario is marked with a `@login` annotation. Edit `features/bootstrap/FeatureContext.php` and add the following methods:

```php
<?php
// features/bootstrap/FeatureContext.php

use App\Entity\User;
use Behat\Behat\Hook\Scope\BeforeScenarioScope;
use Behatch\Context\RestContext;

class FeatureContext implements Context, SnippetAcceptingContext
{
    // ...
    // Must be after createDatabase() and dropDatabase() functions (the order matters)

    /**
     * @BeforeScenario
     * @login
     *
     * @see https://symfony.com/doc/current/security/entity_provider.html#creating-your-first-user
     */
    public function login(BeforeScenarioScope $scope)
    {
        $user = new User();
        $user->setUsername('admin');
        $user->setPassword('ATestPassword');
        $user->setEmail('test@test.com');

        $this->manager->persist($user);
        $this->manager->flush();

        $token = $this->jwtManager->create($user);

        $this->restContext = $scope->getEnvironment()->getContext(RestContext::class);
        $this->restContext->iAddHeaderEqualTo('Authorization', "Bearer $token");
    }

    /**
     * @AfterScenario
     * @logout
     */
    public function logout() {
        $this->restContext->iAddHeaderEqualTo('Authorization', '');
    }
}
```

Then, update `behat.yml` to inject the `lexik_jwt_authentication.jwt_manager`:

```yaml
# behat.yml
default:
  # ...
  suites:
    default:
      contexts:
        - FeatureContext: { doctrine: '@doctrine', 'jwtManager': '@lexik_jwt_authentication.jwt_manager' }
        - Behat\MinkExtension\Context\MinkContext
        - Behatch\Context\RestContext
        - Behatch\Context\JsonContext
  # ...
```

Finally, mark your scenarios with the `@login` annotation to automatically add a valid `Authorization` header, and with `@logout` to be sure to destroy the token after this scenario.

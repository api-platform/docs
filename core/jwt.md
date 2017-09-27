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
<?php

// features/bootstrap/FeatureContext.php

use AppBundle\Entity\User;
use Behat\Behat\Hook\Scope\BeforeScenarioScope;
use Behatch\Context\RestContext;

class FeatureContext implements Context, SnippetAcceptingContext
{
    // ...
    // Must be aster createDatabase() and dropDatabase() functions (the order matters)

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

Then, updates `behat.yml` to inject the `lexik_jwt_authentication.jwt_manager`:

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

Finally, mark your scenarios with the `@login` annotation to automatically add a valid `Authorization` header or with `@logout` to be sure that no user is authenticated.

Previous chapter: [FOSUserBundle Integration](fosuser-bundle.md)

Next chapter: [NelmioApiDocBundle integration](nelmio-api-doc.md)

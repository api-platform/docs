# JWT Authentication with Symfony

> [!NOTE]
> While solutions like `LexikJWTAuthenticationBundle` (Symfony) or `tymondesigns/jwt-auth` (Laravel) are popular,
> **we recommend adopting open standards such as [OpenID Connect (OIDC)](https://openid.net/connect/)** for robust, scalable,
> and interoperable authentication.

<p class="symfonycasts" align="center"><a href="https://symfonycasts.com/screencast/symfony-rest4/json-web-token?cid=apip"><img src="../symfony/images/symfonycasts-player.png" alt="JWT screencast"><br>Watch the LexikJWTAuthenticationBundle screencast</a></p>

## Installing LexikJWTAuthenticationBundle

> [!NOTE]
> API Platform makes it easy to add JWT-based authentication to your API using [LexikJWTAuthenticationBundle](https://github.com/lexik/LexikJWTAuthenticationBundle).

We begin by installing the bundle:

```console
composer require lexik/jwt-authentication-bundle
```
Then we need to generate the public and private keys used for signing JWT tokens.

You can generate them by using this command:

```console
php bin/console lexik:jwt:generate-keypair
```

Or if you're using the [API Platform distribution with Symfony](../symfony/index.md), you may run this from the project's root directory:

```console
docker compose exec php sh -c '
    set -e
    apt-get install openssl
    php bin/console lexik:jwt:generate-keypair
    setfacl -R -m u:www-data:rX -m u:"$(whoami)":rwX config/jwt
    setfacl -dR -m u:www-data:rX -m u:"$(whoami)":rwX config/jwt
'
```

Note that the `setfacl` command relies on the `acl` package. This is installed by default when using the API Platform
docker distribution but may need to be installed in your working environment in order to execute the `setfacl` command.

This takes care of keypair creation (including using the correct passphrase to encrypt the private key), and setting the
correct permissions on the keys allowing the web server to read them.

If you want the keys to be auto generated in `dev` environment, see an example in the
[docker-entrypoint script of api-platform/demo](https://github.com/api-platform/demo/blob/a03ce4fb1f0e072c126e8104e42a938bb840bffc/api/docker/php/docker-entrypoint.sh#L16-L17).

Since these keys are created by the `root` user from a container, your host user will not be able to read them during
the `docker compose build caddy` process. Add the `config/jwt/` folder to the `api/.dockerignore` file so that they are
skipped from the result image.

The keys should not be checked in to the repository (i.e. it's in `api/.gitignore`). However, note that a JWT token could
only pass signature validation against the same pair of keys it was signed with. This is especially relevant in a production
environment, where you don't want to accidentally invalidate all your clients' tokens at every deployment.

For more information, refer to [the bundle's documentation](https://github.com/lexik/LexikJWTAuthenticationBundle/blob/2.x/Resources/doc/index.rst)
or read a [general introduction to JWT here](https://jwt.io/introduction/).

We're not done yet! Let's move on to configuring the Symfony SecurityBundle for JWT authentication.

## Configuring the Symfony SecurityBundle

It is necessary to configure a user provider. You can either use the [Doctrine entity user provider](https://symfony.com/doc/current/security/user_provider.html#entity-user-provider)
provided by Symfony (recommended), [create a custom user provider](https://symfony.com/doc/current/security/user_provider.html#creating-a-custom-user-provider)
or use [API Platform's FOSUserBundle integration](../symfony/fosuser-bundle.md) (**not recommended**).

If you choose to use the Doctrine entity user provider, start by [creating your `User` class](https://symfony.com/doc/current/security.html#a-create-your-user-class).

Then update the security configuration:

```yaml
# api/config/packages/security.yaml
security:
  # https://symfony.com/doc/current/security.html#c-hashing-passwords
  password_hashers:
    App\Entity\User: 'auto'

  # https://symfony.com/doc/current/security.html#where-do-users-come-from-user-providers
  providers:
    # used to reload user from session & other features (e.g. switch_user)
    users:
      entity:
        class: App\Entity\User
        property: email
      # mongodb:
      #    class: App\Document\User
      #    property: email

  firewalls:
    dev:
      pattern: ^/_(profiler|wdt)
      security: false
    main:
      stateless: true
      provider: users
      json_login:
        check_path: auth # The name in routes.yaml is enough for mapping
        username_path: email
        password_path: password
        success_handler: lexik_jwt_authentication.handler.authentication_success
        failure_handler: lexik_jwt_authentication.handler.authentication_failure
      jwt: ~

  access_control:
    - { path: ^/$, roles: PUBLIC_ACCESS } # Allows accessing the Swagger UI
    - { path: ^/docs, roles: PUBLIC_ACCESS } # Allows accessing the Swagger UI docs
    - { path: ^/contexts, roles: PUBLIC_ACCESS } # Allows accessing the Swagger UI contexts
    - { path: ^/auth, roles: PUBLIC_ACCESS }
    - { path: ^/, roles: IS_AUTHENTICATED_FULLY }
```

You must also declare the route used for `/auth`:

```yaml
# api/config/routes.yaml
auth:
  path: /auth
  methods: ['POST']
```

If you want to avoid loading the `User` entity from database each time a JWT token needs to be authenticated, you may consider using
the [database-less user provider](https://github.com/lexik/LexikJWTAuthenticationBundle/blob/2.x/Resources/doc/8-jwt-user-provider.rst) provided by LexikJWTAuthenticationBundle. However, it means you will have to fetch the `User` entity from the database yourself as needed (probably through the Doctrine EntityManager).

Refer to the section on [Security](security.md) to learn how to control access to API resources and operations. You may
also want to [configure Swagger UI for JWT authentication](#documenting-the-authentication-mechanism-with-swaggeropen-api).

### Adding Authentication to an API Which Uses a Path Prefix

If your API uses a [path prefix](https://symfony.com/doc/current/routing/external_resources.html#route-groups-and-prefixes), the security configuration would look something like this instead:

```yaml
# api/config/packages/security.yaml
security:
  # https://symfony.com/doc/current/security.html#c-hashing-passwords
  password_hashers:
    App\Entity\User: 'auto'
  # https://symfony.com/doc/current/security.html#where-do-users-come-from-user-providers
  providers:
    # used to reload user from session & other features (e.g. switch_user)
    users:
      entity:
        class: App\Entity\User
        property: email

  firewalls:
    dev:
      pattern: ^/_(profiler|wdt)
      security: false
    api:
      pattern: ^/api/
      stateless: true
      provider: users
      jwt: ~
    main:
      json_login:
        check_path: auth # The name in routes.yaml is enough for mapping
        username_path: email
        password_path: password
        success_handler: lexik_jwt_authentication.handler.authentication_success
        failure_handler: lexik_jwt_authentication.handler.authentication_failure

  access_control:
    - { path: ^/$, roles: PUBLIC_ACCESS } # Allows accessing the Swagger UI
    - { path: ^/docs, roles: PUBLIC_ACCESS } # Allows accessing API documentations and Swagger UI docs
    - { path: ^/contexts, roles: PUBLIC_ACCESS } # Allows accessing the Swagger UI contexts
    - { path: ^/auth, roles: PUBLIC_ACCESS }
    - { path: ^/, roles: IS_AUTHENTICATED_FULLY }
```

### Be sure to have lexik_jwt_authentication configured on your user_identity_field

```yaml
# api/config/packages/lexik_jwt_authentication.yaml
lexik_jwt_authentication:
  secret_key: '%env(resolve:JWT_SECRET_KEY)%'
  public_key: '%env(resolve:JWT_PUBLIC_KEY)%'
  pass_phrase: '%env(JWT_PASSPHRASE)%'
```

## Documenting the Authentication Mechanism with Swagger/Open API

Want to test the routes of your JWT-authentication-protected API?

### Configuring API Platform

```yaml
# api/config/packages/api_platform.yaml
api_platform:
  swagger:
    api_keys:
      JWT:
        name: Authorization
        type: header
```

The "Authorize" button will automatically appear in Swagger UI.

![Screenshot of API Platform with Authorize button](../core/images/JWTAuthorizeButton.png)

### Adding a New API Key

All you have to do is configure the API key in the `value` field.
By default, [only the authorization header mode is enabled](https://github.com/lexik/LexikJWTAuthenticationBundle/blob/2.x/Resources/doc/index.rst#2-use-the-token) in LexikJWTAuthenticationBundle.
You must set the [JWT token](https://github.com/lexik/LexikJWTAuthenticationBundle/blob/2.x/Resources/doc/index.rst#1-obtain-the-token) as below and click on the "Authorize" button.

`Bearer MY_NEW_TOKEN`

![Screenshot of API Platform with the configuration API Key](../core/images/JWTConfigureApiKey.png)

### Adding endpoint to SwaggerUI to retrieve a JWT token

LexikJWTAuthenticationBundle has an integration with API Platform to automatically
add an OpenAPI endpoint to conveniently retrieve the token in Swagger UI.

If you need to modify the default configuration, you can do it in the dedicated configuration file:

```yaml
# config/packages/lexik_jwt_authentication.yaml
lexik_jwt_authentication:
  # ...
  api_platform:
    check_path: /auth
    username_path: email
    password_path: password
```

You will see something like this in Swagger UI:

![API Endpoint to retrieve JWT Token from SwaggerUI](../core/images/jwt-token-swagger-ui.png)

## Testing

To test your authentication with `ApiTestCase`, you can write a method as below:

```php
<?php
// tests/AuthenticationTest.php

namespace App\Tests;

use ApiPlatform\Symfony\Bundle\Test\ApiTestCase;
use App\Entity\User;
use Hautelook\AliceBundle\PhpUnit\ReloadDatabaseTrait;

class AuthenticationTest extends ApiTestCase
{
    use ReloadDatabaseTrait;

    public function testLogin(): void
    {
        $client = self::createClient();
        $container = self::getContainer();

        $user = new User();
        $user->setEmail('test@example.com');
        $user->setPassword(
            $container->get('security.user_password_hasher')->hashPassword($user, '$3CR3T')
        );

        $manager = $container->get('doctrine')->getManager();
        $manager->persist($user);
        $manager->flush();

        // retrieve a token
        $response = $client->request('POST', '/auth', [
            'headers' => ['Content-Type' => 'application/json'],
            'json' => [
                'email' => 'test@example.com',
                'password' => '$3CR3T',
            ],
        ]);

        $json = $response->toArray();
        $this->assertResponseIsSuccessful();
        $this->assertArrayHasKey('token', $json);

        // test not authorized
        $client->request('GET', '/greetings');
        $this->assertResponseStatusCodeSame(401);

        // test authorized
        $client->request('GET', '/greetings', ['auth_bearer' => $json['token']]);
        $this->assertResponseIsSuccessful();
    }
}
```

Refer to [Testing the API](../symfony/testing.md) for more information about testing API Platform.

### Improving Tests Suite Speed

Since now we have a `JWT` authentication, functional tests require us to log in each time we want to test an API endpoint. This is where [Password Hashers](https://symfony.com/doc/current/security/passwords.html) come into play.

Hashers are used for 2 reasons:

1. To generate a hash for a raw password (`$container->get('security.user_password_hasher')->hashPassword($user, '$3CR3T')`)
2. To verify a password during authentication

While hashing and verifying 1 password is quite a fast operation, doing it hundreds or even thousands of times in a tests suite becomes a bottleneck, because reliable hashing algorithms are slow by their nature.

To significantly improve the test suite speed, we can use more simple password hasher specifically for the `test` environment.

```yaml
# override in api/config/packages/test/security.yaml for test env
security:
  password_hashers:
    App\Entity\User:
      algorithm: md5
      encode_as_base64: false
      iterations: 0
```

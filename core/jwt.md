# JWT Authentification

> [JSON Web Token (JWT)](jwt.io) is a JSON-based open standard ([RFC 7519](https://tools.ietf.org/html/rfc7519)) for creating access tokens that assert some number of claims. For example, a server could generate a token that has the claim "logged in as admin" and provide that to a client. The client could then use that token to prove that he/she is logged in as admin. The tokens are signed by the server's key, so the server is able to verify that the token is legitimate. The tokens are designed to be compact, URL-safe and usable especially in web browser single sign-on (SSO) context.

[Wikipedia](https://en.wikipedia.org/wiki/JSON_Web_Token)

API Platform allows to easily add a JWT-based authentication to your API using [LexikJWTAuthenticationBundle](https://github.com/lexik/LexikJWTAuthenticationBundle).

API Platform is fully working with [LexikJWTAuthenticationBundle](https://github.com/lexik/LexikJWTAuthenticationBundle).

In order to install [the bundle please follow their documentation](https://github.com/lexik/LexikJWTAuthenticationBundle/blob/master/Resources/doc/index.md).

`LexikJWTAuthenticationBundle` requires your application to have a properly configured user provider. You can either use [API Platform's FOSUserBundle integration](fosuser-bundle) or  [create a custom user provider](http://symfony.com/doc/current/security/custom_provider.html).

## Configure Token Acquisition

Here's a sample configuration using FOSUserBundle to secure the API:

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
        dev:
            pattern:  ^/(_(profiler|wdt)|css|images|js)/
            security: false
            
        login:
            pattern:  ^/token
            stateless: true
            anonymous: true
            provider: fos_userbundle
            form_login:
                check_path: /token
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


    access_control:
        - { path: ^/token, role: IS_AUTHENTICATED_ANONYMOUSLY }
        - { path: ^/books, roles: [ ROLE_READER ] }
        - { path: ^/, roles: [ ROLE_READER ] }
```       

## Protect Documentation with same Authentification
You can also add security to the embedded SwaggerUI: 

```yml
# app/config/security.yml

    security:
    firewalls:
        docs:
            pattern: ^/(docs|login|logout)
            form_login:
                provider: fos_userbundle
                default_target_path: api_doc
            logout: true
            anonymous: true

    access_control:
        - { path: ^/login$, role: IS_AUTHENTICATED_ANONYMOUSLY }
        - { path: ^/logout$, role: IS_AUTHENTICATED_ANONYMOUSLY }
```       

Then, you need to provide the login form, and two routes login_check and logout. This configuration is based on FOSUserBundle:

```yml
# app/config/routing.yml

fos_user_security_login:
    path: /login
    defaults: {_controller: 'FOSUserBundle:Security:login'}

fos_user_security_check:
    path: /login_check

app.docs.auth.logout:
    path:     /logout
```       

## Complete Documentation with Auth System
At this state, you can retrieve a token through `/token` URI. But this URI is actually not documented in Swagger. 
To modify documentation, you can register a new `Symfony\Component\Serializer\Normalizer\NormalizerInterface` to overload `ApiPlatform\Core\Swagger\Serializer\DocumentationNormalizer`. 

You can base on this sample : 

```php
//AppBundle/Documentation/JwtDocumentationNormalizer.php

<?php

namespace AppBundle\Documentation;

use Symfony\Component\Serializer\Normalizer\NormalizerInterface;

final class JwtDocumentationNormalizer implements NormalizerInterface
{
    private $normalizerDeferred;

    public function __construct(NormalizerInterface $normalizerDeferred)
    {
        $this->normalizerDeferred = $normalizerDeferred;
    }

    /**
     * {@inheritdoc}
     */
    public function normalize($object, $format = null, array $context = [])
    {
        $TokenDocumentation =[
            'paths' => [
                '/token' => [
                    'post' => [
                        'tags' => ['Token'],
                        'operationId' => 'postTokenItem',
                        'consumes' => ['application/json'],
                        'produces' => ['application/json'],
                        'summary' => 'Get JWT token to login.',
                        'parameters' => [
                            [
                                'in' => 'formData',
                                'name' => 'email',
                                'description' => 'Your Email',
                                'required' => true,
                                'type' => 'string'
                            ],
                            [
                                'in' => 'formData',
                                'name' => 'password',
                                'description' => 'Your password',
                                'required' => true,
                                'type' => 'string'
                            ]
                        ],
                        'responses' => [
                            401 => ['description' => 'Bad credentials'],
                        ],
                    ]
                ]
            ],
            'definitions' => [
                'Token' => [
                    'type' => 'object',
                    'description' => "",
                    'properties' => [
                        'token' => [
                            'type' => 'string',
                            'readOnly' => true,
                        ]
                    ]
                ]
            ]
        ];

        $deferredDocumentation = $this->normalizerDeferred->normalize($object, $format, $context);

        return array_merge_recursive($TokenDocumentation, $deferredDocumentation);
    }

    /**
     * {@inheritdoc}
     */
    public function supportsNormalization($data, $format = null)
    {
        return $this->normalizerDeferred->supportsNormalization($data, $format);
    }
}

```

And register this service like that : 

```yml
# app/config/services.yml

services:
    app.jwt.documentation.normalizer:
      class: 'AppBundle\Documentation\JwtDocumentationNormalizer'
      decorates: api_platform.swagger.normalizer.documentation
      decoration_priority: 2
      arguments: ['@app.jwt.documentation.normalizer.inner']
      public: false
```

Last problem in your documnetation, part of 'Response Messages' does not contain 401 Response (If token is invalid/expire)

This sample add on each route a 401 Response. You can modify this to add your own logic to protect specific route and not each route. 

```php 
//AppBundle/Documentation/Response401DocumentationNormalizer.php

<?php

namespace AppBundle\Documentation;

use Symfony\Component\Serializer\Normalizer\NormalizerInterface;

final class Response401DocumentationNormalizer implements NormalizerInterface
{
    /**
     * @var NormalizerInterface
     */
    private $normalizerDeferred;

    public function __construct(NormalizerInterface $normalizerDeferred)
    {
        $this->normalizerDeferred = $normalizerDeferred;
    }

    /**
     * {@inheritdoc}
     */
    public function normalize($object, $format = null, array $context = [])
    {
        /** @var array[][] $deferredDocumentation */
        $deferredDocumentation = $this->normalizerDeferred->normalize($object, $format, $context);

        $description401 = ['description' => 'Bad credentials'];
        foreach ($deferredDocumentation['paths'] as $deferredPath => $deferredPathDefinition) {
            foreach ($deferredPathDefinition as $deferredMethod => $deferredDefinition) {
                $deferredDocumentation['paths'][$deferredPath][$deferredMethod]['responses'][401] = $description401;
            }
        }

        return $deferredDocumentation;
    }

    /**
     * {@inheritdoc}
     */
    public function supportsNormalization($data, $format = null)
    {
        return $this->normalizerDeferred->supportsNormalization($data, $format);
    }
}
``` 

And register this service like that : 

```yml
# app/config/services.yml

services:
    app.401.documentation.normalizer:
      class: 'AppBundle\Documentation\Response401DocumentationNormalizer'
      decorates: api_platform.swagger.normalizer.documentation
      decoration_priority: 1
      arguments: ['@app.401.documentation.normalizer.inner']
      public: false
```

Now you have a complete documentation and a secure API!


Previous chapter: [FOSUserBundle Integration](fosuser-bundle.md)

Next chapter: [NelmioApiDocBundle integration](nelmio-api-doc.md)

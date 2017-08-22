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
            lexik_jwt: ~

        dev:
            pattern:  ^/(_(profiler|wdt)|css|images|js)/
            security: false

    access_control:
        - { path: ^/login, role: IS_AUTHENTICATED_ANONYMOUSLY }
        - { path: ^/books, roles: [ ROLE_READER ] }
        - { path: ^/, roles: [ ROLE_READER ] }
```       

Previous chapter: [FOSUserBundle Integration](fosuser-bundle.md)

Next chapter: [NelmioApiDocBundle integration](nelmio-api-doc.md)

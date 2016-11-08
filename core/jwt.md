# JWT Authentification


[JWT](https://jwt.io/) is a JSON based way to create access tokens in order to identify the client who wants to access to private information.

This is an exemple in JS that you can use, store the token wherever you want to :

1. POST /login_check
2. localStorage.setItem('token', response.token)
3. headers.add('Authorization', 'Bearer ' + localStorage.getItem(token))

API Platform is fully working with [LexikJWTAuthenticationBundle](https://github.com/lexik/LexikJWTAuthenticationBundle).

In order to install [the bundle please follow their documentation](https://github.com/lexik/LexikJWTAuthenticationBundle/blob/master/Resources/doc/index.md).

Here is a full exemple of a working configuration with a `Person` entity:

```yml
# app/config/security.yml
security:
    encoders:
        AppBundle\Entity\Person: bcrypt

    role_hierarchy:
        ROLE_FOO:   ROLE_USER
        ROLE_ADMIN: ROLE_FOO

    providers:
        app:
            entity:
                class: AppBundle:Person
                property: email

    firewalls:
        login:
            pattern:  ^/login
            stateless: true
            anonymous: true
            form_login:
                check_path:               /login_check
                username_parameter:       email
                password_parameter:       password
                success_handler:          lexik_jwt_authentication.handler.authentication_success
                failure_handler:          lexik_jwt_authentication.handler.authentication_failure
                require_previous_session: false

        api:
            pattern:   ^/
            stateless: true
            anonymous: true
            lexik_jwt: ~

        dev:
            pattern:  ^/(_(profiler|wdt)|css|images|js)/
            security: false

    access_control:
        - { path: ^/login, role: IS_AUTHENTICATED_ANONYMOUSLY }
        - { path: ^/foos, roles: [ ROLE_FOO ] }
        - { path: ^/, roles: [ ROLE_FOO ] }
```       

Previous chapter: [FOSUserBundle Integration](fosuser-bundle.md)

Next chapter: [NelmioApiDocBundle integration](nelmio-api-doc.md)

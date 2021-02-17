# Debugging

<p align="center" class="symfonycasts"><a href="https://symfonycasts.com/screencast/api-platform/profiler?cid=apip"><img src="../distribution/images/symfonycasts-player.png" alt="API Platform debugging screencast"><br>Watch the Debugging API Platform screencast</a></p>

## Xdebug

The default Docker stack is shipped without a Xdebug stage. It's easy
though to add [Xdebug](https://xdebug.org/) to your project, for development
purposes such as debugging tests or remote API requests.

## Add a Development Stage to the Dockerfile

To avoid deploying API Platform to production with an active Xdebug extension,
it's recommended to add a custom stage to the end of the `api/Dockerfile`.

```Dockerfile
# api/Dockerfile
FROM api_platform_php as api_platform_php_dev

ARG XDEBUG_VERSION=3.0.2
RUN set -eux; \
 apk add --no-cache --virtual .build-deps $PHPIZE_DEPS; \
 pecl install xdebug-$XDEBUG_VERSION; \
 docker-php-ext-enable xdebug; \
 apk del .build-deps
```

## Configure Xdebug with Docker Compose Override

Using an [override](https://docs.docker.com/compose/reference/overview/#specifying-multiple-compose-files) file named
`docker-compose.override.yml` ensures that the production configuration remains untouched.

As an example, an override could look like this:

```yml
version: "3.4"

services:
  php:
    build:
      target: api_platform_php_dev
    environment:
      # See https://docs.docker.com/docker-for-mac/networking/#i-want-to-connect-from-a-container-to-a-service-on-the-host
      # See https://github.com/docker/for-linux/issues/264
      # The `remote_host` below may optionally be replaced with `remote_connect_back`
      XDEBUG_CONFIG: >-
        remote_enable=1
        remote_host=host.docker.internal
        remote_connect_back=1
        remote_port=9000
        idekey=PHPSTORM
      # This should correspond to the server declared in PHPStorm `Preferences | Languages & Frameworks | PHP | Servers`
      # Then PHPStorm will use the corresponding path mappings
      PHP_IDE_CONFIG: serverName=api-platform
```

Note for Mac environments use the following:

```yml
      XDEBUG_CONFIG: >-
        remote_enable=1
        remote_host=docker.for.mac.localhost
        remote_connect_back=0
        remote_port=9000
        idekey=PHPSTORM
        remote_autostart=1
        remote_mode=req
        remote_handler=dbgp
```

## Troubleshooting

Inspect the installation with the following command. The requested Xdebug
version should be displayed in the output.

```console
$ docker-compose exec php \
    php --version

PHP …
    with Xdebug v3.0.2, Copyright (c) 2002-2021, by Derick Rethans
    …
```

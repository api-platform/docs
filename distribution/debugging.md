# Debugging

<p align="center" class="symfonycasts"><a href="https://symfonycasts.com/screencast/api-platform/profiler?cid=apip"><img src="/docs/distribution/images/symfonycasts-player.png" alt="API Platform debugging screencast"><br>Watch the Debugging API Platform screencast</a></p>

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

ARG XDEBUG_VERSION=3.1.3
RUN set -eux; \
 apk add --no-cache --virtual .build-deps $PHPIZE_DEPS; \
 pecl install xdebug-$XDEBUG_VERSION; \
 docker-php-ext-enable xdebug; \
 apk del .build-deps
```

## Configure Xdebug with Docker Compose Override

XDebug is shipped by default with the API Platform distribution.
To enable it, run: `XDEBUG_MODE=debug docker compose up --wait`

If you are using VSCode, use the following `launch.json` to debug.
Note that this configuration includes the path mappings for the Docker image.

```json
{
    "version": "0.2.0",
    "configurations": [
        {
            "name": "Listen for Xdebug",
            "type": "php",
            "request": "launch",
            "port": 9003,
            "log": true,
            "pathMappings": {
                "/app": "${workspaceFolder}/api"
            },
        },
    ]
}
```

Note: On Linux, the `client_host` setting of `host.docker.internal` may not work.
In this case you will need the actual local IP address of your computer.

## Troubleshooting

Inspect the installation with the following command. The requested Xdebug
version should be displayed in the output.

```console
$ docker compose exec php \
    php --version

PHP …
    with Xdebug v3.1.3, Copyright (c) 2002-2021, by Derick Rethans
    …
```

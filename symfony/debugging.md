# Debugging

<p align="center" class="symfonycasts"><a href="https://symfonycasts.com/screencast/api-platform/profiler?cid=apip"><img src="images/symfonycasts-player.png" alt="API Platform debugging screencast"><br>Watch the Debugging API Platform screencast</a></p>

## Xdebug

For development purposes such as debugging tests or remote API requests,
[Xdebug](https://xdebug.org/) is shipped by default with the API Platform distribution.

To enable it, run:

```console
XDEBUG_MODE=debug docker compose up --wait
```

## Using Xdebug with PhpStorm

First, [create a PHP debug remote server configuration](https://www.jetbrains.com/help/phpstorm/creating-a-php-debug-server-configuration.html):

1. In the `Settings/Preferences` dialog, go to `PHP | Servers`
2. Create a new server:
   * Name: `api` (or whatever you want to use for the variable `PHP_IDE_CONFIG`)
   * Host: `localhost` (or the one defined using the `SERVER_NAME` environment variable)
   * Port: `443`
   * Debugger: `Xdebug`
   * Check `Use path mappings`
   * Map the local `api/` directory to the `/app` absolute path on the server

You can now use the debugger!

1. In PhpStorm, open the `Run` menu and click on `Start Listening for PHP Debug Connections`
2. Add the `XDEBUG_SESSION=PHPSTORM` query parameter to the URL of the page you want to debug or use [other available triggers](https://xdebug.org/docs/step_debug#activate_debugger).
   Alternatively, you can use [the Xdebug extension](https://xdebug.org/docs/step_debug#browser-extensions) for your preferred web browser.

3. On the command-line, we might need to tell PhpStorm which [path mapping configuration](https://www.jetbrains.com/help/phpstorm/zero-configuration-debugging-cli.html#configure-path-mappings) should be used, set the value of the PHP_IDE_CONFIG environment variable to `serverName=api`, where `api` is the name of the debug server configured higher.

    Example:

    ```console
    XDEBUG_SESSION=1 PHP_IDE_CONFIG="serverName=api" php bin/console ...
    ```

## Using Xdebug With VSCode

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

> [!NOTE]
>
> On Linux, the `client_host` setting of `host.docker.internal` may not work.
> In this case you will need the actual local IP address of your computer.

## Troubleshooting

Inspect the installation with the following command. The requested Xdebug
version should be displayed in the output.

```console
$ docker compose exec php \
    php --version

PHP …
    with Xdebug v…, Copyright (c) 2002-2021, by Derick Rethans
    …
```

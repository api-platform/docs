# Deploying an API Platform App on Heroku

[Heroku](http://heroku.com) is a popular, fast, scalable and reliable *Platform As A Service* (PaaS). As Heroku offers a
free plan including database support trough [Heroku Postgres](https://www.heroku.com/postgres), it's
a very convenient way to experiment with the API Platform.

The API Platform Heroku integration also supports MySQL databases provided by [the ClearDB add-on](https://addons.heroku.com/cleardb).

Deploying API Platform applications on Heroku is very straightforward and you will learn how to do it in this tutorial.

*Note: this tutorial works perfectly well with API Platform but also with any Symfony application based on the Symfony Standard
Edition.*

If you haven't already one, [create an account on Heroku](https://signup.heroku.com/signup/dc). Then install [the Heroku
toolbelt](https://devcenter.heroku.com/articles/getting-started-with-php#local-workstation-setup). We guess you already
have a working install of [Composer](http://getcomposer.org), perfect, we will need it.

Create a new API Platform project as usual:

    composer create-project api-platform/api-platform

Go to the created directory. Then install the API Heroku integration library created by the API Platform team. It we will ease the deployment.
Install it:

    composer require dunglas/api-platform-heroku

Heroku relies on [environment variables](https://devcenter.heroku.com/articles/config-vars) for configuration. Independently of the provider you
choose for hosting your application, using environment variables to configure your production environment is a best practice.
So we will configure the library we just installed and remove the Incenteev Parameter Handler library that was bundled with
API Platform. Parameter Handler generated the `app/config/parameters.yml` file during the installation process.

Open `composer.json` file and remove the following line in the  `require` section:

```json
"incenteev/composer-parameter-handler": "~2.0",
```

Then remove the following script call in the `post-install-cmd` and `post-update-cmd` sections:

```json
"Incenteev\\ParameterHandler\\ScriptHandler::buildParameters",
```

Then we must register the Composer script provided by the library we installed in the `scripts` section of the `composer.json`
file:

```json
    "scripts": {
        "pre-install-cmd": [
          "Dunglas\\Heroku\\Database::createParameters"
        ],
        // ...
    }
```

Delete `app/config/parameters.yml` and `app/config/parameters.yml.dist` as they will not be used anymore. The remove the
the following line from the `imports` section of `app/config/config.yml`:

```yaml
    - { resource: parameters.yml }
```

We will now create a Heroku the `app.json` file at the root of the application directory to set the parameters of our application
using the external parameters feature of the Symfony container:

```json
{
  "success_url": "/",
  "env": {
    "SYMFONY_ENV": "prod",
    "SYMFONY__DATABASE_DRIVER": "pdo_pgsql",
    "SYMFONY__MAILER_TRANSPORT": "smtp",
    "SYMFONY__MAILER_HOST": "your-mailer.com",
    "SYMFONY__MAILER_USER": "your-mailer-username",
    "SYMFONY__MAILER_PASSWORD": "your-mailer-password",
    "SYMFONY__CORS_ALLOW_ORIGIN": "https://your-client-url.com",
    "SYMFONY__LOCALE": "en",
    "SYMFONY__SECRET": {
      "generator": "secret"
    }
  },
  "addons": [
    "heroku-postgresql"
  ],
  "buildpacks": [
    {
      "url": "https://github.com/heroku/heroku-buildpack-php"
    }
  ]
}
```

The file also tell to the Heroku deployment system to build a PHP container and to add the Postgres add-on.

If you also want to run your app locally or on another hosting provider, don't forget to set those environment variables
and another one called `DATABASE_URL` containing your database DSN.
A convenient way to manage environment variable is the [PHP dotenv](https://github.com/vlucas/phpdotenv) library.

We are almost done, but API Platform (and Symfony) have a particular directory structure, the document root is `web/`, other
directory must be private and we must tell it to Heroku. Create a new file at the root of the application directory named
`Procfile` with the following content:

```yaml
web: bin/heroku-php-apache2 web/
```

Our application is ready to be deployed, but Heroku dynos are not persistent and file stored directly on the filesystem
will be lost. It's problematic for our logs.

Note: if you want to store files permanently, use a persistent file storage service such as Amazon S3.

Heroku provides another free service called  https://devcenter.heroku.com/articles/logplex allowing to centralize and
persist applications logs. To use it we need to configure Monolog to output logs on `STDERR` instead of in a file.

Open `app/config/config_prod.yml`, find the following block:

```yaml
    monolog:
        # ...
        nested:
            type:  stream
            path:  "%kernel.logs_dir%/%kernel.environment%.log"
            level: debug
```

And replace it by:

```yaml
    monolog:
        # ...
        nested:
            type:  stream
            path:  "php://stderr"
            level: debug
```

We are ready to deploy our app!

Initialize a git repository:

    git init

Add all existing files:

    git add --all

Commit:

    git commit -a -m "My first API Platform app running on Heroku!"

Create the Heroku application:

    heroku create

And deploy for the first time:

    git push heroku master

Your browser should open automatically and display the entrypoint of the API. It's time to create the database:

    heroku run "app/console doctrine:schema:create"

**We're done.** You can play with the demo bookstore API provided with API Platform. It is ready for production and you
can scale it in one click from the Heroku interface.

To see your logs, run `heroku logs --tail`.

Can it be easier? Yes it can: we are preparing an API Platform edition preconfigured to run on Heroku! Stay tuned.

Previous chapter: [Introduction](index.md)
Next chapter: [Using API Platform with Docker](docker.md)

# Deploying an API Platform App on Heroku

[Heroku](https://www.heroku.com) is a popular, fast, scalable and reliable *Platform As A Service* (PaaS). As Heroku offers a
free plan including database support through [Heroku Postgres](https://www.heroku.com/postgres), it's a convenient way
to experiment with API Platform.

The API Platform Heroku integration also supports MySQL databases provided by [the ClearDB add-on](https://addons.heroku.com/cleardb).

Deploying API Platform applications on Heroku is straightforward and you will learn how to do it in this tutorial.

*Note: this tutorial works perfectly well with API Platform but also with any Symfony application based on the Symfony Standard
Edition.*

If you don't already have one, [create an account on Heroku](https://signup.heroku.com/signup/dc). Then install [the Heroku
toolbelt](https://devcenter.heroku.com/articles/getting-started-with-php#set-up). We're guessing you already
have a working install of [Composer](http://getcomposer.org). Perfect, we will need it.

Create a new [API Platform project](distribution/index.md) which will be used in the rest of this example.

Heroku relies on [environment variables](https://devcenter.heroku.com/articles/config-vars) for its configuration. Regardless
of what provider you choose for hosting your application, using environment variables to configure your production environment
is a best practice promoted by API Platform.

Create a Heroku `app.json` file at the root of the `api/` directory to configure the deployment:

```json
{
  "success_url": "/",
  "env": {
    "APP_ENV": "prod",
    "APP_SECRET": {"generator": "secret"},
    "CORS_ALLOW_ORIGIN": "https://your-client-url.com"
  },
  "addons": [
    "heroku-postgresql"
  ],
  "buildpacks": [
    {
      "url": "https://github.com/heroku/heroku-buildpack-php"
    }
  ],
  "scripts": {
    "postdeploy": "php bin/console doctrine:schema:create"
  }
}
```

The file also tells the Heroku deployment system to build a PHP container and to add the Postgres add-on.

We are almost done, but API Platform (and Symfony) has a particular directory structure which requires further configuration.
We must tell Heroku that the document root is `public/`, and that all other directories must be private.

Create a new file named `Procfile` in the `api/` directory with the following content:

```yaml
web: vendor/bin/heroku-php-apache2 public/
```

Be sure to add the Apache Pack to your dependencies:

```console
composer require symfony/apache-pack
```

As Heroku doesn't support Varnish out of the box, let's disable its integration:

```diff
# config/packages/api_platform.yaml
-    http_cache:
-        invalidation:
-            enabled: true
-            varnish_urls: ['%env(VARNISH_URL)%']
-        max_age: 0
-        shared_max_age: 3600
-        vary: ['Content-Type', 'Authorization', 'Origin']
-        public: true
```

Heroku provides another free service, [Logplex](https://devcenter.heroku.com/articles/logplex), which allows us to centralize
and persist application logs. Because API Platform writes logs on `STDERR`, it will work seamlessly.

However, if you use Monolog instead of the default logger, you'll need to configure it to output to `STDERR` instead of
in a file.

Open `config/packages/prod/monolog.yaml` and apply the following patch:

```diff
     handlers:
         nested:
             type: stream
-            path: "%kernel.logs_dir%/%kernel.environment%.log"
+            path: php://stderr
             level: debug
```

We are now ready to deploy our app!

Go to the `api/` directory, then

1. Initialize a git repository:

    git init

2. Add all existing files:

    git add --all

3. Commit:

    git commit -a -m "My first API Platform app running on Heroku!"

4. Create the Heroku application:

    heroku create

5. And deploy for the first time:

    git push heroku master

**We're done.** You can play with the demo API provided with API Platform. It is ready for production and you
can scale it in one click from the Heroku interface.

To see your logs, run `heroku logs --tail`.

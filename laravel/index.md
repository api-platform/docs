# API Platform for Laravel Projects

API Platform is **the easiest way** to create **state of the art** web APIs
using Laravel!

With API Platform, you can:

* expose your Eloquent models in minutes as:
  * a REST API implementing the industry-leading standards and best practices: JSON-LD, JSON:API and HAL
  * a GraphQL API
  * or both at the same time, with the same code!
* automatically expose an OpenAPI specification (formerly Swagger), dynamically generated from your Eloquent models and always up to date
* automatically expose nice UIs and playgrounds to develop using your API (Swagger UI, Redoc, GraphiQL and/or GraphQL Playground)
* automatically paginate your collections
* add validation logic using the built-in Laravel logic
* add authorization logic using gates and policies
* add filtering logic
* push changed data to the clients in real-time using Mercure and Broadcast
* benefits from the API Platform JavaScript tools: admin and client generator (supports Next/React, Nuxt/Vue.js, Quasar, Vuetify and more!)
* benefits from native HTTP cache (with automatic invalidation)
* boot your app with Laravel Octane and FrankenPHP (a blazing fast PHP app server created by the author of API Platform)
* decouple your API from your models and implement patterns such as CQRS
* test your API using convenient ad-hoc assertions that works with Pest and PHPUnit

Let's discover how to use API Platform with Laravel!

## Install Laravel

API Platform can be installed easily on new and existing Laravel projects.
If you already have an existing project, skip directly to the next section.

If you don't have an existing Laravel project, [create one](https://laravel.com/docs/11.x/installation).
All Laravel installation methods are supported. For instance, you can use Composer:

```console
composer create-project laravel/laravel my-api-platform-laravel-app
cd my-api-platform-laravel-app
```

## Install API Platform

In your Laravel project, install the API Platform integration for Laravel:

```console
composer require api-platform/laravel:^4@alpha
```

If it's not already done, run `php artisan serve` to start the built-in web server.

Open http://127.0.0.1:8000/api/docs! Your API is already active and documented... but empty!

![Empty docs](images/empty-docs.png)

## Create an Eloquent Model


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
* add validation logic using Laravel logic
* add authorization logic using gates and policies (compatible with Passport and Sanctum)
* add filtering logic
* push changed data to the clients in real-time using Laravel Broadcast and Mercure (a popular WebSockets alternative, created by the authors of API Platform) and receive them using Laravel Ech
* benefits from the API Platform JavaScript tools: [admin](../admin/index.md) and [create client](../create-client/index.md) (supports Next/React, Nuxt/Vue.js, Quasar, Vuetify and more!)
* benefits from native HTTP cache (with automatic invalidation)
* boot your app with [Octane](https://laravel.com/docs/octane) and [FrankenPHP](https://frankenphp.dev) (the default Octane engine, also created by the authors of API Platform)
* decouple your API from your models and implement patterns such as CQRS
* test your API using convenient ad-hoc assertions that works with Pest and PHPUnit

Let's discover how to use API Platform with Laravel!

## Install Laravel

API Platform can be installed easily on new and existing Laravel projects.
If you already have an existing project, skip directly to the next section.

If you don't have an existing Laravel project, [create one](https://laravel.com/docs/installation).
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

Open http://127.0.0.1:8000/api/, your API is already active and documented... but empty!

![Empty docs](images/empty-docs.png)

## Create an Eloquent Model

To discover how API Platform framework works, we will create an API to manage a bookshop.

Let's start by creating a `Book` model:

```console
php artisan make:model Book
```

By default, Laravel uses SQLite. You can open the `database/database.sqlite` file with your prefered SQLite client (PHPStorm works like charm), create a table named `books` and add some columns, Eloquent and API Platform will detect these columns automatically.

But there is a better alternative: using a migration class.

### Create a Migration

First, create a migration class for the `books` table:

```console
 php artisan make:migration create_books_table
```

Open the generated migration class (`database/migrations/<timestamp>_create_books_table.php`) and add some columns:

```patch
    public function up(): void
    {
        Schema::create('books', function (Blueprint $table) {
            $table->id();

+            $table->string('isbn')->nullable();
+            $table->string('title');
+            $table->string('description');
+            $table->string('author');
+            $table->date('publication_date')->nullable();

            $table->timestamps();
        });
    }
```

Finally execute the migration:

```console
php artisan migrate
```

The table and columns have been created for you!

## Expose Your Model

Open `app/Models/Book.php` that we generated in the previous step and mark the class it contains with the `#[ApiResource]` attribute:

```patch
namespace App\Models;

+use ApiPlatform\Metadata\ApiResource;
use Illuminate\Database\Eloquent\Model;

+#[ApiResource]
class Book extends Model
{
}
```

Open http://127.0.0.1:8000/api/, tadam, your API is ready and **entirely functionnal** 🎉:

![Basic REST API](images/basic-rest.png)

You can play with your API with the sandbox provided by SwaggerUI.

Under the hood, API Platform:

1. Registered the standard REST routes in Laravel's router and a controller that implements a state-of-the-art, fully-featured and secure API endpoint using the services provided by the [API Platform Core library](../core/index.md)
2. Used its built-in Eloquent [state provider](../core/state-providers.md) to introspect the database and gather metadata about all columns to expose through the API
3. Generated machine readable documentations of the API in the [OpenAPI (formerly known as Swagger)](https://www.openapis.org/) (available at http://127.0.0.1:8000/api/docs.json) and [JSON-LD](https://json-ld.org/)/[Hydra](https://www.hydra-cg.com/) formats using these metadata
4. Generated a nice human-readable documentation and a sandbox for the API with [SwaggerUI](https://swagger.io/tools/swagger-ui/) (Redoc is also available out of the box)

Imagine doing it all again, properly, by hand? How much time have you saved? Weeks, months? And you've seen nothing yet!

## Populate the Database

```php
<?php

namespace Database\Factories;

use Illuminate\Database\Eloquent\Factories\Factory;

/**
 * @extends \Illuminate\Database\Eloquent\Factories\Factory<\App\Models\Book>
 */
class BookFactory extends Factory
{
    /**
     * Define the model's default state.
     *
     * @return array<string, mixed>
     */
    public function definition(): array
    {
        return [
            'title' => fake()->title(),
            'isbn' => fake()->isbn13(),
            'description' => fake()->text(),
            'author' => fake()->userName(),
            'publication_date' => fake()->date()
        ];
    }
}
```

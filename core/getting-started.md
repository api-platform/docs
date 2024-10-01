# Getting started

## Installing API Platform

You can choose your preferred stack between Symfony, Laravel, or bootstrapping the API Platform core library manually.

> [!CAUTION]
> If you are migrating from an older version of API Platform, make sure you read the [Upgrade Guide](upgrade-guide.md).

### Symfony

If you are starting a new project, the easiest way to get API Platform up is to install [API Platform for Symfony](../symfony/index.md).

It comes with the API Platform core library integrated with [the Symfony framework](https://symfony.com), [the schema generator](../schema-generator/),
[Doctrine ORM](https://www.doctrine-project.org),
[NelmioCorsBundle](https://github.com/nelmio/NelmioCorsBundle) and [test assertions dedicated to APIs](testing.md).

[MongoDB](mongodb.md) and [Elasticsearch](elasticsearch.md) can also be easily enabled.

Basically, it is a Symfony edition packaged with the best tools to develop a REST and GraphQL APIs and sensible default settings.

Alternatively, you can use [Composer](https://getcomposer.org/) to install the standalone bundle in an existing Symfony Flex
project:

```console
composer require API
```

There are no mandatory configuration options although [many settings are available](configuration.md).

### Migrating from FOSRestBundle

If you plan to migrate from FOSRestBundle, you might want to read [this guide](migrate-from-fosrestbundle.md) to get started with API Platform.

### Laravel

API Platform can be installed on any new or existing Laravel project using [API Platform for Laravel](../laravel/index.md).

It comes with integrations from the Laravel ecosystem, including [Eloquent](https://laravel.com/docs/eloquent), [Validation](https://laravel.com/docs/validation), [Authorization](https://laravel.com/docs/authorization), [Octane](https://laravel.com/docs/octane), [Pest](https://pestphp.com)...

### Bootstrapping the Core Library

While more complex, the core library [can also be installed in vanilla PHP projects and other frameworks](../core/bootstrap.md).

## Before Reading this Documentation

If you haven't read it already, take a look at [the Laravel Getting Started guide](../laravel/index.md) or [the Symfony Getting Started guide](../symfony/index.md).
These tutorials cover basic concepts required to understand how API Platform works including how it implements the REST architectural style
and what [JSON-LD](https://json-ld.org/) and [Hydra](https://www.hydra-cg.com/) formats are.

## Mapping the Entities

### Symfony with Doctrine

<p align="center" class="symfonycasts"><a href="https://symfonycasts.com/screencast/api-platform/api-resource?cid=apip"><img src="../symfony/images/symfonycasts-player.png" alt="Create an API Resource screencast"><br>Watch the Create an API Resource screencast</a></p>

API Platform can automatically expose entities mapped as "API resources" through a REST API supporting CRUD
operations.
To expose your entities, you can use attributes, XML, and YAML configuration files.

Here is an example of entities mapped using attributes that will be exposed through a REST API:

```php
<?php
// api/src/Entity/Product.php
namespace App\Entity;

use ApiPlatform\Metadata\ApiResource;
use Doctrine\ORM\Mapping as ORM;
use Doctrine\Common\Collections\ArrayCollection;
use Symfony\Component\Validator\Constraints as Assert;

#[ORM\Entity]
#[ApiResource]
class Product // The class name will be used to name exposed resources
{
    #[ORM\Id, ORM\Column, ORM\GeneratedValue]
    private ?int $id = null;

    /**
     * A name property - this description will be available in the API documentation too.
     *
     */
    #[ORM\Column] 
    #[Assert\NotBlank]
    public string $name = '';

    // Notice the "cascade" option below, this is mandatory if you want Doctrine to automatically persist the related entity
    /**
     * @var Offer[]|ArrayCollection
     *
     */
    #[ORM\OneToMany(targetEntity: Offer::class, mappedBy: 'product', cascade: ['persist'])] 
    public iterable $offers;

    public function __construct()
    {
        $this->offers = new ArrayCollection(); // Initialize $offers as a Doctrine collection
    }

    public function getId(): ?int
    {
        return $this->id;
    }

    // Adding both an adder and a remover as well as updating the reverse relation is mandatory
    // if you want Doctrine to automatically update and persist (thanks to the "cascade" option) the related entity
    public function addOffer(Offer $offer): void
    {
        $offer->product = $this;
        $this->offers->add($offer);
    }

    public function removeOffer(Offer $offer): void
    {
        $offer->product = null;
        $this->offers->removeElement($offer);
    }

    // ...
}
```

```php
<?php
// api/src/Entity/Offer.php
namespace App\Entity;

use ApiPlatform\Metadata\ApiResource;
use Doctrine\ORM\Mapping as ORM;
use Symfony\Component\Validator\Constraints as Assert;

/**
 * An offer from my shop - this description will be automatically extracted from the PHPDoc to document the API.
 *
 */
#[ORM\Entity]
#[ApiResource(types: ['https://schema.org/Offer'])]
class Offer
{
    #[ORM\Id, ORM\Column, ORM\GeneratedValue]
    private ?int $id = null;

    #[ORM\Column(type: 'text')]
    public string $description = '';

    #[ORM\Column]
    #[Assert\Range(minMessage: 'The price must be superior to 0.', min: 0)]
    public float $price = -1.0;

    #[ORM\ManyToOne(targetEntity: Product::class, inversedBy: 'offers')]
    public ?Product $product = null;

    public function getId(): ?int
    {
        return $this->id;
    }
}
```

It is the minimal configuration required to expose `Product` and `Offer` entities as JSON-LD documents through an hypermedia
web API.

If you are familiar with the Symfony ecosystem, you noticed that entity classes are also mapped with Doctrine ORM attributes
and validation constraints from [the Symfony Validator Component](https://symfony.com/doc/current/validation.html).
This isn't mandatory. You can use [your preferred persistence](state-providers.md) and [validation](validation.md) systems.
However, API Platform has built-in support for those libraries and is able to use them without requiring any specific
code or configuration to automatically persist and validate your data. They are a good default option and we encourage you to use
them unless you know what you are doing.

Thanks to the mapping done previously, API Platform will automatically register the following REST [operations](operations.md)
for resources of the product type:

### Product API using Symfony

Method | URL            | Description
-------|----------------|--------------------------------
GET    | /products      | Retrieve the (paginated) collection
POST   | /products      | Create a new product
GET    | /products/{id} | Retrieve a product
PATCH  | /products/{id} | Apply a partial modification to a product
DELETE | /products/{id} | Delete a product

> [!NOTE]
>
> `PUT` (replace or create) isn't registered automatically,
> but is entirely supported by API Platform and can be added explicitly.
The same operations are available for the offer method (routes will start with the `/offers` pattern).
Route prefixes are built by pluralizing the name of the mapped entity class.
It is also possible to override the naming convention using [operation path namings](operation-path-naming.md).

As an alternative to attributes, you can map entity classes using YAML or XML:

<code-selector>

```yaml
# api/config/api_platform/resources.yaml
resources:
    App\Entity\Product: ~
    App\Entity\Offer:
        shortName: 'Offer'                   # optional
        description: 'An offer from my shop' # optional
        types: ['https://schema.org/Offer']   # optional
        paginationItemsPerPage: 25           # optional
```
```xml
<?xml version="1.0" encoding="UTF-8" ?>
<!-- api/config/api_platform/resources.xml -->

<resources xmlns="https://api-platform.com/schema/metadata/resources-3.0"
        xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:schemaLocation="https://api-platform.com/schema/metadata/resources-3.0
        https://api-platform.com/schema/metadata/resources-3.0.xsd">
    <resource class="App\Entity\Product" />
    <resource
        class="App\Entity\Offer"
        shortName="Offer" <!-- optional -->
        description="An offer from my shop" <!-- optional -->
    >
        <types>
            <type>https://schema.org/Offer</type> <!-- optional -->
        </types>
    </resource>
</resources>
```

</code-selector>

If you prefer to use YAML or XML files instead of attributes, you must configure API Platform to load the appropriate files:

```yaml
# api/config/packages/api_platform.yaml
api_platform:
    mapping:
        paths: 
            - '%kernel.project_dir%/src/Entity' # default configuration for attributes
            - '%kernel.project_dir%/config/api_platform' # yaml or xml directory configuration
```
If you want to serialize only a subset of your data, please refer to the [Serialization documentation](serialization.md).
**You're done!**
You now have a fully featured API exposing your entities.
Run the Symfony app with the [Symfony Local Web Server](https://symfony.com/doc/current/setup/symfony_server.html) (`symfony server:start`) and browse the API entrypoint at `http://localhost:8000/api`.

### Laravel with Eloquent

API Platform introspects the database (column names, types, constraints, types, constraints...) to populate API Platform metadata.
Serialization, OpenAPI, and hydra docs are generated from these metadata directly.

#### Example

First, create a migration class for the `products` table:

```console
 php artisan make:migration create_products_table
```

Open the generated migration class (`database/migrations/<timestamp>_create_products_table.php`) and add some columns:

```patch
    public function up(): void
    {
        Schema::create('products', function (Blueprint $table) {
            $table->id();

+            $table->string('name');
+            $table->decimal('price', 8, 2);
+            $table->text('description');
+            $table->boolean('is_active')->default(true);
+            $table->date('created_date')->nullable();

            $table->timestamps();
        });
    }
```

Finally, execute the migration:

```console
php artisan
```

And after that, just adding the `#[ApiResource]` attribute as follows onto your model:

```patch
<?php

namespace App\Models;
//app/Models/Product.php
+use ApiPlatform\Metadata\ApiResource;
use Illuminate\Database\Eloquent\Model;
 
+#[ApiResource]
class Product extends Model {}
```

While attributes (introduced in PHP 8) are the preferred way to configure your API Platform resources, it’s also possible to use a trait instead.
```patch
<?php

namespace App\Models;
//app/Models/Product.php
+use ApiPlatform\Metadata\IsApiResource;
use Illuminate\Database\Eloquent\Model;
 
class Product extends Model 
{
+   use IsApiResource;
}
```

See the [Using The IsApiResourceTrait Instead of Attributes documentation](../laravel/index.md#using-the-isapiresourcetrait-instead-of-attributes) for examples of advanced configuration in both cases (attributes or static method).

**You're done!** 🎉

You now have a fully featured API exposing your entities.

Run the Laravel app with the Laravel's local development server using [the Artisan Console component](https://laravel.com/docs/artisan) (`php artisan serve`) and browse the API entrypoint at `http://localhost:8000/api`.

Using configurations, API Platform generates metadata that will automatically register the following REST [operations](operations.md)
for resources of the product type:

### Product API using Laravel

Method | URL            | Description
-------|----------------|--------------------------------
GET    | /products      | Retrieve the (paginated) collection
POST   | /products      | Create a new product
GET    | /products/{id} | Retrieve a product
PATCH  | /products/{id} | Apply a partial modification to a product
DELETE | /products/{id} | Delete a product

In addition, among other things, API Platform under the hood does the following:
- Generated machine-readable documentations of the API in the [OpenAPI (formerly known as Swagger)](../core/openapi.md) (available at `http://127.0.0.1:8000/api/docs.json`) and [JSON-LD](https://json-ld.org)/[Hydra](https://www.hydra-cg.com) formats using this metadata
- Generated nice human-readable documentation and a sandbox for the API with [SwaggerUI](https://swagger.io/tools/swagger-ui/) (Redoc is also available out-of-the-box)

## Interactions with the API

Interact with the API using a REST client (we recommend [Hoppscotch](https://hoppscotch.io/)) or [API Platform Admin](../admin/index.md).

## Examples

Take a look at [the API Platform demo](https://github.com/api-platform/demo).

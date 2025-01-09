# Getting started

## Migrating from FOSRestBundle

If you plan to migrate from FOSRestBundle, you might want to read [this guide](migrate-from-fosrestbundle.md) to get started with API Platform.

## Installing API Platform Core

If you are starting a new project, the easiest way to get API Platform up is to install the [API Platform Distribution](../distribution/index.md).
It comes with the API Platform Core library integrated with [the Symfony framework](https://symfony.com), [the schema generator](../schema-generator/),
[Doctrine ORM](https://www.doctrine-project.org), [Elasticsearch-PHP](https://www.elastic.co/guide/en/elasticsearch/client/php-api/current/index.html),
[NelmioCorsBundle](https://github.com/nelmio/NelmioCorsBundle) and [Behat](http://behat.org).
[Doctrine MongoDB ODM](https://www.doctrine-project.org/projects/mongodb-odm.html) can also be enabled by following the [MongoDB documentation](mongodb.md).
Basically, it is a Symfony edition packaged with the best tools to develop a REST API and sensible default settings.

Alternatively, you can use [Composer](http://getcomposer.org) to install the standalone bundle in an existing Symfony Flex
project:

`composer require api`

There are no mandatory configuration options although [many settings are available](configuration.md).

## Before Reading this Documentation

If you haven't read it already, take a look at [the Getting Started guide](../distribution/index.md).
This tutorial covers basic concepts required to understand how API Platform works including how it implements the REST pattern
and what [JSON-LD](http://json-ld.org/) and [Hydra](http://www.hydra-cg.com/) formats are.

## Mapping the Entities

<p align="center" class="symfonycasts"><a href="https://symfonycasts.com/screencast/api-platform/api-resource?cid=apip"><img src="../distribution/images/symfonycasts-player.png" alt="Create an API Resource screencast"><br>Watch the Create an API Resource screencast</a></p>

API Platform Core is able to automatically expose entities mapped as "API resources" through a REST API supporting CRUD
operations.
To expose your entities, you can use Docblock annotations, XML and YAML configuration files.

Here is an example of entities mapped using annotations which will be exposed through a REST API:

```php
<?php
// api/src/Entity/Product.php

namespace App\Entity;

use ApiPlatform\Core\Annotation\ApiResource;
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

use ApiPlatform\Core\Annotation\ApiResource;
use Doctrine\ORM\Mapping as ORM;
use Symfony\Component\Validator\Constraints as Assert;

/**
 * An offer from my shop - this description will be automatically extracted from the PHPDoc to document the API.
 *
 */
#[ORM\Entity]
#[ApiResource(iri: 'https://schema.org/Offer')]
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

If you are familiar with the Symfony ecosystem, you noticed that entity classes are also mapped with Doctrine ORM annotations
and validation constraints from [the Symfony Validator Component](http://symfony.com/doc/current/book/validation.html).
This isn't mandatory. You can use [your preferred persistence](data-providers.md) and [validation](validation.md) systems.
However, API Platform Core has built-in support for those libraries and is able to use them without requiring any specific
code or configuration to automatically persist and validate your data. They are a good default option and we encourage you to use
them unless you know what you are doing.

Thanks to the mapping done previously, API Platform Core will automatically register the following REST [operations](operations.md)
for resources of the product type:

### Product

Method | URL            | Description
-------|----------------|--------------------------------
GET    | /products      | Retrieve the (paginated) collection
POST   | /products      | Create a new product
GET    | /products/{id} | Retrieve a product
PUT    | /products/{id} | Update a product
PATCH  | /products/{id} | Apply a partial modification to a product
DELETE | /products/{id} | Delete a product

The same operations are available for the offer method (routes will start with the `/offers` pattern).
Route prefixes are built by pluralizing the name of the mapped entity class.
It is also possible to override the naming convention using [operation path namings](operation-path-naming.md).

As an alternative to annotations, you can map entity classes using YAML or XML:

<code-selector>

```yaml
# config/api_platform/resources.yaml
resources:
    App\Entity\Product: ~
    App\Entity\Offer:
        shortName: 'Offer'                   # optional
        description: 'An offer from my shop' # optional
        iri: 'https://schema.org/Offer'       # optional
        attributes:                          # optional
            pagination_items_per_page: 25    # optional
```

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<!-- api/config/api_platform/resources.xml -->

<resources xmlns="https://api-platform.com/schema/metadata"
        xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:schemaLocation="https://api-platform.com/schema/metadata
        https://api-platform.com/schema/metadata/metadata-2.0.xsd">
    <resource class="App\Entity\Product" />
    <resource
        class="App\Entity\Offer"
        shortName="Offer" <!-- optional -->
        description="An offer from my shop" <!-- optional -->
        iri="https://schema.org/Offer" <!-- optional -->
    />
</resources>
```

</code-selector>

If you prefer to use YAML or XML files instead of annotations, you must configure API Platform to load the appropriate files:

```yaml
# config/packages/api_platform.yaml
api_platform:
    mapping:
        paths: 
            - '%kernel.project_dir%/src/Entity' # default configuration for annotations
            - '%kernel.project_dir%/config/api_platform' # yaml or xml directory configuration
```

If you want to serialize only a subset of your data, please refer to the [Serialization documentation](serialization.md).

**You're done!**

You now have a fully featured API exposing your entities.
Run the Symfony app with the [Symfony Local Web Server](https://symfony.com/doc/current/setup/symfony_server.html) (`symfony server:start`) and browse the API entrypoint at `http://localhost:8000/api`.

Interact with the API using a REST client (we recommend [Postman](https://www.getpostman.com/)) or an Hydra-aware application
(you should give [Hydra Console](https://github.com/lanthaler/HydraConsole) a try). Take
a look at the usage examples in [the `features` directory](https://github.com/api-platform/core/tree/main/features).

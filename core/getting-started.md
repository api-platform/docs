# Getting started

## Installing API Platform Core

If you are starting a new project, the easiest way to get API Platform up is to install the [API Platform Distribution](../distribution/index.md).
It ships with the API Platform Core library integrated with [the Symfony framework](https://symfony.com), [the schema generator](../schema-generator/),
[Doctrine ORM](http://www.doctrine-project.org), [NelmioCorsBundle](https://github.com/nelmio/NelmioCorsBundle)
and [Behat](http://behat.org).
Basically, it is a Symfony edition packaged with the best tools to develop a REST API and sensible default settings.

Alternatively, you can use [Composer](http://getcomposer.org) to install the standalone bundle in an existing Symfony Flex
project:

`composer require api`

There is no mandatory configuration options although [many settings are available](configuration.md).

## Before Reading this Documentation

If you haven't read it already, take a look at [the Getting Started guide](../distribution/index.md).
This tutorial covers basic concepts required to understand how API Platform works including how it implements the REST pattern
and what [JSON-LD](http://json-ld.org/) and [Hydra](http://www.hydra-cg.com/) formats are.

## Mapping the Entities

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

/**
 * @ApiResource
 * @ORM\Entity
 */
class Product // The class name will be used to name exposed resources
{
    /**
     * @ORM\Column(type="integer")
     * @ORM\Id
     * @ORM\GeneratedValue(strategy="AUTO")
     */
    public $id;

    /**
     * @var string $name A name property - this description will be available in the API documentation too.
     *
     * @ORM\Column
     * @Assert\NotBlank
     */
    public $name;

    // Notice the "cascade" option below, this is mandatory if you want Doctrine to automatically persist the related entity
    /**
     * @ORM\OneToMany(targetEntity="Offer", mappedBy="product", cascade={"persist"})
     */
    public $offers;

    public function __construct()
    {
        $this->offers = new ArrayCollection(); // Initialize $offers as an Doctrine collection
    }

    // Adding both an adder and a remover as well as updating the reverse relation are mandatory
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
 * An offer from my shop - this description will be automatically extracted form the PHPDoc to document the API.
 *
 * @ApiResource(iri="http://schema.org/Offer")
 * @ORM\Entity
 */
class Offer
{
    /**
     * @ORM\Column(type="integer")
     * @ORM\Id
     * @ORM\GeneratedValue(strategy="AUTO")
     */
    public $id;

    /**
     * @ORM\Column(type="text")
     */
    public $description;

    /**
     * @ORM\Column(type="float")
     * @Assert\NotBlank
     * @Assert\Range(min=0, minMessage="The price must be superior to 0.")
     * @Assert\Type(type="float")
     */
    public $price;

    /**
     * @ORM\ManyToOne(targetEntity="Product", inversedBy="offers")
     */
    public $product;
}
```

It is the minimal configuration required to expose `Product` and `Offer` entities as JSON-LD documents through an hypermedia
web API.

If you are familiar with the Symfony ecosystem, you noticed that entity classes are also mapped with Doctrine ORM annotations
and validation constraints from [the Symfony Validator Component](http://symfony.com/doc/current/book/validation.html).
This isn't mandatory. You can use [your preferred persistence](data-providers.md) and [validation](validation.md) systems.
However, API Platform Core has built-in support for those library and is able to use them without requiring any specific
code or configuration to automatically persist and validate your data. They are good default and we encourage you to use
them unless you know what you are doing.

Thanks to the mapping done previously, API Platform Core will automatically register the following REST [operations](operations.md)
for resources of the product type:

Product

Method | URL            | Description
-------|----------------|--------------------------------
GET    | /products      | Retrieve the (paged) collection
POST   | /products      | Create a new product
GET    | /products/{id} | Retrieve a product
PUT    | /products/{id} | Update a product
DELETE | /products/{id} | Delete a product

The same operations are available for the offer method (routes will start with the `/offers` pattern).
Route prefixes are built by pluralizing the name of the mapped entity class.
It is also possible to override the naming convention using [operation path namings](operation-path-naming.md).

As an alternative to annotations, you can map entity classes using XML or YAML:

XML:

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
        description="An offer form my shop" <!-- optional -->
        iri="http://schema.org/Offer" <!-- optional -->
    />
</resources>
```

YAML:

```yaml
# api/config/api_platform/resources.yaml
resources:
    App\Entity\Product: ~
    App\Entity\Offer:
        shortName: 'Offer'                   # optional
        description: 'An offer from my shop' # optional
        iri: 'http://schema.org/Offer'       # optional
        attributes:                          # optional
            pagination_items_per_page: 25    # optional
```

**You're done!**

You now have a fully featured API exposing your entities.
Run the Symfony app (`bin/console server:run`) and browse the API entrypoint at `http://localhost:8000/api`.

Interact with the API using a REST client (we recommend [Postman](https://www.getpostman.com/)) or an Hydra aware application
(you should give [Hydra Console](https://github.com/lanthaler/HydraConsole) a try). Take
a look at the usage examples in [the `features` directory](https://github.com/api-platform/core/tree/master/features).

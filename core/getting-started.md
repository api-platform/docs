# Getting started

## Installing API Platform Core

If you are starting a new project, the easiest way to get API Platform up is to install the [API Platform Standard Edition](https://github.com/api-platform/api-platform).
It ships with the API Platform Core library integrated with [the Symfony framework](https://symfony.com), [the schema generator](../schema-generator/),
[Doctrine ORM](www.doctrine-project.org), [NelmioApiDocBundle](https://github.com/nelmio/NelmioApiDocBundle), [NelmioCorsBundle](https://github.com/nelmio/NelmioCorsBundle)
and [Behat](http://behat.org).
Basically, it is a Symfony edition packaged with the best tools to develop a REST API and sensitive default settings.

Alternatively, you can use [Composer](http://getcomposer.org) to install the standalone bundle in an existing Symfony project:

`composer require api-platform/core`

Then, update your `app/config/AppKernel.php` file:

```php
// app/config/AppKernel.php

public function registerBundles()
{
    $bundles = [
        // ...
        new ApiPlatform\Core\Bridge\Symfony\Bundle\ApiPlatformBundle(),
        // ...
    ];

    // ...
}
```

Register the routes of our API by adding the following lines to `app/config/routing.yml`:

```yaml
api:
    resource: '.'
    type:     'api_platform'
    prefix:   '/api' # Optional
```

There is no mandatory configuration. However, a bunch of [configuration parameters](configuration.md) are available.

## Before Reading this Documentation

If you haven't read it already, take a look at [the "Creating your first API with API Platform, in 5 minutes" guide](../tutorial/api.md).
Using the schema generator is not necessary to use API Platform Core. But the "Exposing the API" section of this tutorial
covers basic concepts required to understand how API Platform works including how it implements the REST pattern and what
[JSON-LD](http://json-ld.org/) and [Hydra](http://www.hydra-cg.com/) formats are.

## Mapping the Entities

API Platform Core is able to automatically expose entities mapped as "API resources" through a REST API supporting CRUD operations.
To expose your entities, you can use Docblock annotations, XML and YAML configuration files.

Here is an example of entities mapped using annotations which will be exposed trough a REST API:

```php
<?php

// src/AppBundle/Entity/Product.php

namespace AppBundle\Entity;

use ApiPlatform\Core\Annotation\ApiResource;
use Doctrine\ORM\Mapping as ORM;
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
     * @param string $name A name property - this description will be avaliable in the API documentation too.
     *
     * @ORM\Column
     * @Assert\NotBlank
     */
    public $name;
}
```

```php
<?php

// src/AppBundle/Entity/Offer.php

namespace AppBundle\Entity;

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
     * @ORM\ManyToOne(targetEntity="Product")
     */
    public $product;
}
```

It is the minimal configuration required to expose `Product` and `Offer` entities as JSON-LD documents trough an hypermedia
web API.

If you are familiar with the Symfony ecosystem, you noticed that entity classes are also mapped with Doctrine ORM annotations
and validation constraints from [the Symfony Validator Component](http://symfony.com/doc/current/book/validation.html).
This isn't mandatory. You can use [your preferred persistence](data-providers.md) and [validation](the-event-system.md) systems.
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
Routes prefixes are built by pluralizing the name of the mapped entity class.
It is also possible to override the naming convention using [operation path namings](operation-path-naming.md).

As an alternative to annotations, you can map entity classes using XML or YAML:

<configurations>

```xml
# src/AppBundle/Resources/config/resources.xml
<?xml version="1.0" encoding="UTF-8" ?>
<resources>
    <resource class="AppBundle\Entity\Product" />
    <resource
        class="AppBundle\Entity\Offer"
        shortName="Offer" <!-- optional -->
        description="An offer form my shop" <!-- optional -->
        iri="http://schema.org/Offer" <!-- optional -->
    />
</resources>
```

```yaml
# src/AppBundle/Resources/config/resources.yml

resources:
    product:
        class: 'AppBundle\Entity\Product'
    offer:
        class: 'AppBundle\Entity\Offer'
        shortName: 'Offer' # optional        # optional
        description: 'An offer from my shop' # optional
```

</configurations>

**You're done!**

You now have a fully featured API exposing your entities.
Run the Symfony app (`bin/console server:run`) and browse the API entrypoint at `http://localhost:8000/api`.

Interact with the API using a REST client (we recommend [Postman](https://www.getpostman.com/)) or an Hydra aware application
(you should give a try to [Hydra Console](https://github.com/lanthaler/HydraConsole)). Take
a look at the usage examples in [the `features` directory](/features/).

Previous chapter: [Introduction](index.md)
Next chapter: [Configuration](configuration.md)

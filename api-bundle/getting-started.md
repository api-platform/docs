# Getting started

## Installing ApiPlatformBundle

If you are starting a new project, the easiest way to get ApiPlatformBundle up, running and well integrated with other useful
tools including [PHP Schema](http://php-schema.dunglas.com), [NelmioApiDocBundle](https://github.com/nelmio/NelmioApiDocBundle),
[NelmioCorsBundle](https://github.com/nelmio/NelmioCorsBundle) and [Behat](http://behat.org) is to install [API Platform Standard Edition](https://github.com/api-platform/api-platform).
It's a Symfony edition packaged with the best tools to develop a REST API and sensitive default settings.

Alternatively, you can use [Composer](http://getcomposer.org) to install the standalone bundle in your project:

`composer require dunglas/api-bundle`

Then, update your `app/config/AppKernel.php` file:

```php
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
```

## Configuring the API

### Minimal configuration

The first step is to name your API. Add the following lines in `app/config/config.yml`:

```yaml
api_platform:
    title:                             'My Dummy API'
    description:                       'This is a test API.'
```

The name and the description you give will be accessible through the auto-generated Hydra documentation.

### Full configuration

Here's the complete configuration with the default:

```yaml
# Default configuration for extension with alias: "api_platform"
api_platform:
    title:                             'My Dummy API'
    description:                       'This is a test API.'
    supported_formats:
        jsonld:                        ['application/ld+json']
        xml:                           ['application/xml', 'text/xml']
    name_converter:                    'app.name_converter'
    enable_fos_user:                   true
    collection:
        order_parameter_name:          'order'
        order:                         'ASC'
        pagination:
            client_enabled:            true
            client_items_per_page:     true
            items_per_page:            3
```

The name and the description you give will be accessible through the auto-generated Hydra documentation.

## Mapping the entities

Imagine you have the following Doctrine entity classes:

```php
<?php

// src/AppBundle/Entity/Product.php

namespace AppBundle\Entity;

use Doctrine\ORM\Mapping as ORM;
use Symfony\Component\Validator\Constraints as Assert;

/**
 * @ORM\Entity
 */
class Product
{
    /**
     * @ORM\Column(type="integer")
     * @ORM\Id
     * @ORM\GeneratedValue(strategy="AUTO")
     */
    public $id;

    /**
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

use Doctrine\ORM\Mapping as ORM;
use Symfony\Component\Validator\Constraints as Assert;

/**
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

## Registering the services

Register the following services (for example in `app/config/services.yml`):

```yaml
services:
    resource.product:
        parent:    "api.resource"
        arguments: [ "AppBundle\Entity\Product" ]
        tags:      [ { name: "api.resource" } ]

    resource.offer:
        parent:    "api.resource"
        arguments: [ "AppBundle\Entity\Offer" ]
        tags:      [ { name: "api.resource" } ]
```

**You're done!**

You now have a fully featured API exposing your Doctrine entities.
Run the Symfony app (`bin/console server:run`) and browse the API entrypoint at `http://localhost:8000/api`.

Interact with the API using a REST client (I recommend [Postman](https://chrome.google.com/webstore/detail/postman-rest-client/fdmmgilgnpjigdojojpjoooidkmcomcm))
or an Hydra aware application (you should give a try to [Hydra Console](https://github.com/lanthaler/HydraConsole)). Take
a look at the usage examples in [the `features` directory](/features/).

Next chapter: [NelmioApiDocBundle integration](nelmio-api-doc.md)

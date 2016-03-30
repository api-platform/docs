# Getting started

## Installing API Platform

If you are starting a new project, the easiest way to get API Platform up is to install [API Platform Standard Edition](https://github.com/api-platform/api-platform). It ships with very convenient tools such as [the schema generator](../schema-generator/), [NelmioApiDocBundle](https://github.com/nelmio/NelmioApiDocBundle),
[NelmioCorsBundle](https://github.com/nelmio/NelmioCorsBundle) and [Behat](http://behat.org).
It's a Symfony edition packaged with the best tools to develop a REST API and sensitive default settings.

Alternatively, you can use [Composer](http://getcomposer.org) to install the standalone bundle in your project:

`composer require api-platform/core`

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
    type:     'api'
    prefix:   '/api' # Optional
```

## Configuring the API

### Minimal configuration

The first step is to name your API. Add the following lines in `app/config/config.yml`:

```yaml
api_platform:
    title:       'Your API name'                    # The title of the API.
    description: 'The full description of your API' # The description of the API.
```

The name and the description you give will be accessible through the auto-generated Hydra documentation.

### Full configuration

Here's the complete configuration with the default:

```yaml
api_platform:

    # The title of the API.
    title:                ~ # Required

    # The description of the API.
    description:          ~ # Required

    # The list of enabled formats. The first one will be the default.
    supported_formats:

        # Prototype
        format:
            mime_types:           []

    # Specify a name converter to use.
    name_converter:       null

    # Enable the FOSUserBundle integration.
    enable_fos_user:      false

    # Enable the Nelmio Api doc integration.
    enable_nelmio_api_doc:  true
    collection:

        # The default order of results.
        order:                null

        # The name of the query parameter to order results.
        order_parameter_name:  order
        pagination:

            # To enable or disable pagination for all resource collections by default.
            enabled:              true

            # To allow the client to enable or disable the pagination.
            client_enabled:       false

            # To allow the client to set the number of items per page.
            client_items_per_page:  false

            # The default number of items per page.
            items_per_page:       30

            # The default name of the parameter handling the page number.
            page_parameter_name:  page

            # The name of the query parameter to enable or disable pagination.
            enabled_parameter_name:  pagination

            # The name of the query parameter to set the number of items per page.
            items_per_page_parameter_name:  itemsPerPage
    metadata:
        resource:

            # Cache service for resource metadata.
            cache:                api_platform.metadata.resource.cache.array
        property:

            # Cache service for property metadata.
            cache:                api_platform.metadata.property.cache.array
```

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

## Registering resources

A resource can be defined through Annotations, Yaml or XML. The following represents the minimal configuration to register a resource for Product and one for Offer. A resource is then represented by REST endpoints called [Operations](operations.md).

<configurations>

```php
<?php
// src/AppBundle/Entity/Product.php

namespace AppBundle\Entity;

use ApiPlatform\Core\Annotation\Resource;
use Doctrine\ORM\Mapping as ORM;
use Symfony\Component\Validator\Constraints as Assert;

/**
 * @ORM\Entity
 * @Resource
 */
class Product
{
//...
}

// src/AppBundle/Entity/Offer.php

namespace AppBundle\Entity;

use ApiPlatform\Core\Annotation\Resource;
use Doctrine\ORM\Mapping as ORM;
use Symfony\Component\Validator\Constraints as Assert;

/**
 * @ORM\Entity
 * @Resource
 */
class Offer
{
//...
}
```

```yaml
# src/AppBundle/Resources/config/resources.yml
resources:
  product:
    class: 'AppBundle\Entity\Product'
  offer:
    class: 'AppBundle\Entity\Offer'
```

</configurations>

**You're done!**

You now have a fully featured API exposing your Doctrine entities.
Run the Symfony app (`bin/console server:run`) and browse the API entrypoint at `http://localhost:8000/api`.

Interact with the API using a REST client (we recommend [Postman](https://chrome.google.com/webstore/detail/postman-rest-client/fdmmgilgnpjigdojojpjoooidkmcomcm))
or an Hydra aware application (you should give a try to [Hydra Console](https://github.com/lanthaler/HydraConsole)). Take
a look at the usage examples in [the `features` directory](/features/).

Next chapter: [NelmioApiDocBundle integration](nelmio-api-doc.md)

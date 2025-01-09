# MongoDB Support

## Overview

[MongoDB](https://www.mongodb.com/) is one of the most popular NoSQL document-oriented database, used for its high
write load (useful for analytics or IoT) and high availability (easy to set replica sets with automatic failover). It
can also shard the database easily for horizontal scalability and has a powerful query language for doing aggregation,
text search or geospatial queries.

API Platform uses [Doctrine MongoDB ODM 2](https://www.doctrine-project.org/projects/mongodb-odm.html) and in particular
its [aggregation builder](https://www.doctrine-project.org/projects/doctrine-mongodb-odm/en/latest/reference/aggregation-builder.html)
to leverage all the possibilities of the database.

Doctrine MongoDB ODM 2 relies on the [mongodb](https://secure.php.net/manual/en/set.mongodb.php) PHP extension and not on
the legacy [mongo](https://secure.php.net/manual/en/book.mongo.php) extension.

## Enabling MongoDB Support

If the `mongodb` PHP extension is not installed yet, [install it beforehand](https://secure.php.net/manual/en/mongodb.installation.pecl.php).

If you are using the [API Platform Distribution](../distribution/index.md), modify the `Dockerfile` to add the extension:

```diff
# Dockerfile
  pecl install \
   apcu-${APCU_VERSION} \
+  mongodb \
  ; \
  pecl clear-cache; \
  docker-php-ext-enable \
   apcu \
+  mongodb \
   opcache \
  ; \
```

Then rebuild the `php` image:

```console
docker-compose build php
```

Add a MongoDB image to the docker-compose file:

```yaml
# docker-compose.yml
# ...
  db-mongodb:
      # In production, you may want to use a managed database service
      image: mongo
      environment:
          - MONGO_INITDB_DATABASE=api
          - MONGO_INITDB_ROOT_USERNAME=api-platform
          # You should definitely change the password in production
          - MONGO_INITDB_ROOT_PASSWORD=!ChangeMe!
      volumes:
          - db-data:/var/lib/mongodb/data:rw
          # You may use a bind-mounted host directory instead, so that it is harder to accidentally remove the volume and lose all your data!
          # - ./docker/db/data:/var/lib/mongodb/data:rw
      ports:
          - "27017:27017"
# ...
```

Once the extension is installed, to enable the MongoDB support, require the [Doctrine MongoDB ODM bundle](https://github.com/doctrine/DoctrineMongoDBBundle)
package using Composer:

```console
docker-compose exec php \
    composer require doctrine/mongodb-odm-bundle
```

Execute the contrib recipe to have it already configured.

Change the MongoDB environment variables to match your Docker image:

```shell
# .env
MONGODB_URL=mongodb://api-platform:!ChangeMe!@db-mongodb
MONGODB_DB=api
```

Change the configuration of API Platform to add the right mapping path:

```yaml
# config/packages/api_platform.yaml
api_platform:
    # ...

    mapping:
        paths: ['%kernel.project_dir%/src/Entity', '%kernel.project_dir%/src/Document']

    # ...
```

## Creating Documents

Creating resources mapped to MongoDB documents is as simple as creating entities:

```php
<?php
// api/src/Document/Product.php

namespace App\Document;

use ApiPlatform\Core\Annotation\ApiResource;
use Doctrine\Common\Collections\ArrayCollection;
use Doctrine\ODM\MongoDB\Mapping\Annotations as ODM;
use Symfony\Component\Validator\Constraints as Assert;

/**
 * @ApiResource
 *
 * @ODM\Document
 */
class Product
{
    /**
     * @ODM\Id(strategy="INCREMENT", type="int")
     */
    private $id;

    /**
     * @ODM\Field
     */
    #[Assert\NotBlank]
    public $name;

    /**
     * @ODM\ReferenceMany(targetDocument=Offer::class, mappedBy="product", cascade={"persist"}, storeAs="id")
     */
    public $offers;

    public function __construct()
    {
        $this->offers = new ArrayCollection();
    }

    public function getId(): ?int
    {
        return $this->id;
    }

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
// api/src/Document/Offer.php

namespace App\Document;

use ApiPlatform\Core\Annotation\ApiResource;
use Doctrine\ODM\MongoDB\Mapping\Annotations as ODM;
use Symfony\Component\Validator\Constraints as Assert;

/**
 * @ODM\Document
 */
#[ApiResource(iri: "https://schema.org/Offer")]
class Offer
{
    /**
     * @ODM\Id(strategy="INCREMENT", type="int")
     */
    private $id;

    /**
     * @ODM\Field
     */
    public $description;

    /**
     * @ODM\Field(type="float")
     * @Assert\NotBlank
     * @Assert\Range(min=0, minMessage="The price must be superior to 0.")
     * @Assert\Type(type="float")
     */
    public $price;

    /**
     * @ODM\ReferenceOne(targetDocument=Product::class, inversedBy="offers", storeAs="id")
     */
    public $product;

    public function getId(): ?int
    {
        return $this->id;
    }
}
```

When defining references, always use the ID for storing them instead of the native [DBRef](https://docs.mongodb.com/manual/reference/database-references/#dbrefs).
It allows API Platform to manage [filtering on nested properties](filters.md#apifilter-annotation) by using [lookups](https://docs.mongodb.com/manual/reference/operator/aggregation/lookup/).

## Filtering

Doctrine MongoDB ODM filters are practically the same as Doctrine ORM filters.

See how to use them and how to create custom ones in the [filters documentation](filters.md).

## Creating Custom Extensions

See how to create Doctrine MongoDB ODM custom extensions in the [extensions documentation](extensions.md).

## Adding Execute Options

If you want to add some command options when executing an aggregate query (see the [related documentation in MongoDB manual](https://docs.mongodb.com/manual/reference/command/aggregate/#command-fields)),
you can do it in your resource configuration, at the operation or the resource level.

For instance at the operation level:

```php
<?php
// api/src/Document/Offer.php

namespace App\Document;

use ApiPlatform\Core\Annotation\ApiResource;
use Doctrine\ODM\MongoDB\Mapping\Annotations as ODM;

/**
 * @ODM\Document
 */
#[ApiResource(attributes: [
    collectionOperations => [
        "get" => [
            "method" => "GET",
            "doctrine_mongodb" => [
                "execute_options" => [
                    "allowDiskUse" => true,
                ]
            ]
        ]
    ]
])]
class Offer
{
    // ...
}
```

Or at the resource level:

```php
<?php
// api/src/Document/Offer.php

namespace App\Document;

use ApiPlatform\Core\Annotation\ApiResource;
use Doctrine\ODM\MongoDB\Mapping\Annotations as ODM;

/**
 * @ODM\Document
 */
#[ApiResource(attributes: [
    "doctrine_mongodb" => [
        "execute_options" => [
            "allowDiskUse" => true
        ]
    ]
])]
class Offer
{
    // ...
}
```

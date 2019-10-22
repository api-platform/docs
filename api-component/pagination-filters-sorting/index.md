# Introduction

API Platform Core provides a generic system to apply filters and sort criteria on collections.
Useful filters for Doctrine ORM, MongoDB ODM and ElasticSearch are provided with the library.

Here's the curated list of built-in filters, ready to use:

* [Search Filter](search.md)
* [Date Filter](date.md)
* [Range Filter](range.md)
* [Numeric Filter](numeric.md)
* [Boolean Filter](boolean.md)
* [Exists Filter](exists.md)
* [Match Filter](match.md) (ElasticSearch)
* [Order Filter](order.md)
* [Group Filter](group.md)
* [Property Filter](property.md)

You can also [create custom filters](custom.md) that fit your specific needs.
You can also add filtering support to your custom [data providers](../fetching-and-persisting-data/data-providers.md) by implementing interfaces provided
by the library.

**By default, all filters are disabled. They must be enabled explicitly.**

When a filter is enabled, it automatically appears in the [OpenAPI](../documenting-specifying-your-api/swagger.md) and [GraphQL](../graphql/index.md) documentations.
It is also automatically documented as a `hydra:search` property for JSON-LD responses.

<p align="center" class="symfonycasts"><a href="https://symfonycasts.com/screencast/api-platform/json-ld?cid=apip"><img src="../../distribution/images/symfonycasts-player.png" alt="Filtering and Searching screencast"><br>Watch the Filtering & Searching screencast</a>

## Basic Knowledge

Filters are services (see the section on [custom filters](custom.md), and they can be linked to a Resource in two ways:

### 1. Through the `ApiResource` declaration, as the `filters` attribute.

For example having a filter service declaration:

```yaml
# api/config/services.yaml
services:
    # ...
    offer.date_filter:
        parent: 'api_platform.doctrine.orm.date_filter'
        arguments: [ { dateProperty: ~ } ]
        tags:  [ 'api_platform.filter' ]
        # The following are mandatory only if a _defaults section is defined
        # You may want to isolate filters in a dedicated file to avoid adding them
        autowire: false
        autoconfigure: false
        public: false
```

We're linking the filter `offer.date_filter` with the `@ApiResource` annotation:

```php
<?php
// api/src/Entity/Offer.php

namespace App\Entity;

use ApiPlatform\Core\Annotation\ApiResource;

/**
 * @ApiResource(attributes={"filters"={"offer.date_filter"}})
 */
class Offer
{
    // ...
}
```

Alternatively, using YAML:

```yaml
# api/config/api_platform/resources.yaml
App\Entity\Offer:
    collectionOperations:
        get:
            filters: ['offer.date_filter']
    # ...
```

Or XML:

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<!-- api/config/api_platform/resources.xml -->

<resources xmlns="https://api-platform.com/schema/metadata"
           xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
           xsi:schemaLocation="https://api-platform.com/schema/metadata
           https://api-platform.com/schema/metadata/metadata-2.0.xsd">
    <resource class="App\Entity\Offer">
        <collectionOperations>
            <collectionOperation name="get">
                <attribute name="filters">
                    <attribute>offer.date_filter</attribute>
                </attribute>
            </collectionOperation>
            <!-- ... -->
        </collectionOperations>
    </resource>
</resources>
```

### 2. By using the `@ApiFilter` annotation.

This annotation automatically declares the service, and you just have to use the filter class you want:

```php
<?php
// api/src/Entity/Offer.php

namespace App\Entity;

use ApiPlatform\Core\Annotation\ApiFilter;
use ApiPlatform\Core\Annotation\ApiResource;
use ApiPlatform\Core\Bridge\Doctrine\Orm\Filter\DateFilter;

/**
 * @ApiResource
 * @ApiFilter(DateFilter::class, properties={"dateProperty"})
 */
class Offer
{
    // ...
}
```

Learn more on how the [ApiFilter annotation](../pagination-filters-sorting/index.md#the-apifilter-annotation) works.

For the sake of consistency, we're using the annotation in the below documentation.

## The ApiFilter Annotation

The annotation can be used on a `property` or on a `class`.

If the annotation is given over a property, the filter will be configured on the property. For example, let's add a search filter on `name` and on the `prop` property of the `colors` relation:

```php
<?php

use ApiPlatform\Core\Annotation\ApiFilter;
use ApiPlatform\Core\Annotation\ApiResource;
use ApiPlatform\Core\Bridge\Doctrine\Orm\Filter\SearchFilter;
use Doctrine\ORM\Mapping as ORM;

/**
 * @ApiResource
 */
class DummyCar
{
    /**
     * @ORM\Id
     * @ORM\GeneratedValue
     * @ORM\Column(type="integer")
     */
    private $id;

    /**
     * @ORM\Column(type="string")
     * @ApiFilter(SearchFilter::class, strategy="partial")
     */
    public $name;

    /**
     * @ORM\OneToMany(targetEntity="DummyCarColor", mappedBy="car")
     * @ApiFilter(SearchFilter::class, properties={"colors.prop": "ipartial"})
     */
    public $colors;

    // ...
}

```

On the first property, `name`, it's straightforward. The first annotation argument is the filter class, the second specifies options, here, the strategy:

```
@ApiFilter(SearchFilter::class, strategy="partial")
```

In the second annotation, we specify `properties` on which the filter should apply. It's necessary here because we don't want to filter `colors` but the `prop` property of the `colors` association.
Note that for each given property we specify the strategy:

```
@ApiFilter(SearchFilter::class, properties={"colors.prop": "ipartial"})
```

The `ApiFilter` annotation can be set on the class as well. If you don't specify any properties, it'll act on every property of the class.

For example, let's define three data filters (`DateFilter`, `SearchFilter` and `BooleanFilter`) and two serialization filters (`PropertyFilter` and `GroupFilter`) on our `DummyCar` class:

```php
<?php
// api/src/Entity/DummyCar.php

namespace App\Entity;

use ApiPlatform\Core\Annotation\ApiFilter;
use ApiPlatform\Core\Annotation\ApiResource;
use ApiPlatform\Core\Bridge\Doctrine\Orm\Filter\BooleanFilter;
use ApiPlatform\Core\Bridge\Doctrine\Orm\Filter\DateFilter;
use ApiPlatform\Core\Bridge\Doctrine\Orm\Filter\SearchFilter;
use ApiPlatform\Core\Serializer\Filter\GroupFilter;
use ApiPlatform\Core\Serializer\Filter\PropertyFilter;
use Doctrine\ORM\Mapping as ORM;

/**
 * @ApiResource
 * @ORM\Entity
 * @ApiFilter(BooleanFilter::class)
 * @ApiFilter(DateFilter::class, strategy=DateFilter::EXCLUDE_NULL)
 * @ApiFilter(SearchFilter::class, properties={"colors.prop": "ipartial", "name": "partial"})
 * @ApiFilter(PropertyFilter::class, arguments={"parameterName": "foobar"})
 * @ApiFilter(GroupFilter::class, arguments={"parameterName": "foobargroups"})
 */
class DummyCar
{
    // ...
}

```

The `BooleanFilter` is applied to every `Boolean` property of the class. Indeed, in each core filter we check the Doctrine type. It's written only by using the filter class:

```
@ApiFilter(BooleanFilter::class)
```

The `DateFilter` given here will be applied to every `Date` property of the `DummyCar` class with the `DateFilter::EXCLUDE_NULL` strategy:

```
@ApiFilter(DateFilter::class, strategy=DateFilter::EXCLUDE_NULL)
```

The `SearchFilter` here adds properties. The result is the exact same as the example with annotations on properties:

```
@ApiFilter(SearchFilter::class, properties={"colors.prop": "ipartial", "name": "partial"})
```

Note that you can specify the `properties` argument on every filter.

The next filters are not related to how the data is fetched but rather to how the serialization is done on those. We can give an `arguments` option ([see here for the available arguments](#serializer-filters)):

```
@ApiFilter(PropertyFilter::class, arguments={"parameterName": "foobar"})
@ApiFilter(GroupFilter::class, arguments={"parameterName": "foobargroups"})
```

## Enabling a Filter for All Properties of a Resource

As we have seen in previous examples, properties where filters can be applied must be explicitly declared. If you don't
care about security and performance (e.g. an API with restricted access), it is also possible to enable built-in filters
for all properties:

```php
<?php
// api/src/Entity/Offer.php

namespace App\Entity;

use ApiPlatform\Core\Annotation\ApiFilter;
use ApiPlatform\Core\Annotation\ApiResource;
use ApiPlatform\Core\Bridge\Doctrine\Orm\Filter\OrderFilter;

/**
 * @ApiResource
 * @ApiFilter(OrderFilter::class)
 */
class Offer
{
    // ...
}
```

**Note: Filters on [nested properties](nested-properties.md) must still be enabled explicitly, in order to keep things sane.**

Regardless of this option, filters can be applied on a property only if:

* the property exists
* the value is supported (ex: `asc` or `desc` for the order filters).

It means that the filter will be **silently** ignored if the property:

* does not exist
* is not enabled
* has an invalid value



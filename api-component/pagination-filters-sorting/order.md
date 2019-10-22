# Order Filter (Sorting)

The order filter allows to sort a collection against the given properties.

Syntax: `?order[property]=<asc|desc>`

Of course the name of this parameter can be changed per resource (see below) or [globally](#using-a-custom-order-query-parameter-name).

## Implementations

* Doctrine ORM: 
    * Filter class: `ApiPlatform\Core\Bridge\Doctrine\Orm\Filter\OrderFilter`
* Doctrine MongoDB ODM: 
    * Filter class: `ApiPlatform\Core\Bridge\Doctrine\MongoDbOdm\Filter\OrderFilter`
* ElasticSearch: 
    * Filter class: `ApiPlatform\Core\Bridge\Elasticsearch\DataProvider\Filter\OrderFilter`

## Usage

To enable sorting on your resource collection, enable the `OrderFilter`:

```php
<?php
// api/src/Entity/Offer.php

namespace App\Entity;

use ApiPlatform\Core\Annotation\ApiFilter;
use ApiPlatform\Core\Annotation\ApiResource;
use ApiPlatform\Core\Bridge\Doctrine\Orm\Filter\OrderFilter; 
// or ApiPlatform\Core\Bridge\Doctrine\MongoDbOdm\Filter\OrderFilter for MongoDB users

/**
 * @ApiResource
 * @ApiFilter(OrderFilter::class, properties={"id", "name"}, arguments={"orderParameterName"="order"})
 */
class Offer
{
    // ...
}
```

Given that the collection endpoint is `/offers`, you can filter offers by name in ascending order and then by ID in descending
order with the following query: `/offers?order[name]=desc&order[id]=asc`.

## Default order

By default, whenever the query does not specify the direction explicitly (e.g.: `/offers?order[name]&order[id]`), filters
will not be applied unless you configure a default order direction to use:

```php
<?php
// api/src/Entity/Offer.php

namespace App\Entity;

use ApiPlatform\Core\Annotation\ApiFilter;
use ApiPlatform\Core\Annotation\ApiResource;
use ApiPlatform\Core\Bridge\Doctrine\Orm\Filter\OrderFilter;

/**
 * @ApiResource
 * @ApiFilter(OrderFilter::class, properties={"id": "ASC", "name": "DESC"})
 */
class Offer
{
    // ...
}
```



## Using a Custom Order Query Parameter Name

A conflict will occur if `order` is also the name of a property with the search filter enabled.
Luckily, the query parameter name to use is configurable:

```yaml
# api/config/packages/api_platform.yaml
api_platform:
    collection:
        order_parameter_name: '_order' # the URL query parameter to use is now "_order"
```

## Overriding Default Order

API Platform Core provides an easy way to override the default order of items in your collection.

By default, items in the collection are ordered in ascending (ASC) order by their resource identifier(s). If you want to
customize this order, you must add an `order` attribute on your ApiResource annotation:

```php
<?php
// api/src/Entity/Book.php

namespace App\Entity;

use ApiPlatform\Core\Annotation\ApiResource;

/**
 * @ApiResource(attributes={"order"={"foo": "ASC"}})
 */
class Book
{
    // ...

    /**
     * ...
     */
    public $foo;
    
    // ...
}
```

This `order` attribute is used as an array: the key defines the order field, the values defines the direction.
If you only specify the key, `ASC` direction will be used as default. For example, to order by `foo` & `bar`:

```php
<?php
// api/src/Entity/Book.php

namespace App\Entity;

use ApiPlatform\Core\Annotation\ApiResource;

/**
 * @ApiResource(attributes={"order"={"foo", "bar"}})
 */
class Book
{
    // ...

    /**
     * ...
     */
    public $foo;

    /**
     * ...
     */
    public $bar;
    
    // ...
}
```

It's also possible to configure the default order on an association property:

```php
<?php
// api/src/Entity/Book.php

namespace App\Entity;

use ApiPlatform\Core\Annotation\ApiResource;

/**
 * @ApiResource(attributes={"order"={"author.username"}})
 */
class Book
{
    // ...

    /**
     * @var User
     */
    public $author;
    
    // ...
}
```


### Comparing with Null Values

_Note: This section only applies on Doctrine ORM and Doctrine MongoDB ODM._

When the property used for ordering can contain `null` values, you may want to specify how `null` values are treated in
the comparison:

Description                          | Strategy to set
-------------------------------------|---------------------------------------------------------------------------------------------
Use the default behavior of the DBMS | `null`
Consider items as smallest           | `ApiPlatform\Core\Bridge\Doctrine\Orm\Filter\OrderFilter::NULLS_SMALLEST` (`nulls_smallest`)
Consider items as largest            | `ApiPlatform\Core\Bridge\Doctrine\Orm\Filter\OrderFilter::NULLS_LARGEST` (`nulls_largest`)

For instance, treat entries with a property value of `null` as the smallest, with the following service definition:


```php
<?php
// api/src/Entity/Offer.php

namespace App\Entity;

use ApiPlatform\Core\Annotation\ApiFilter;
use ApiPlatform\Core\Annotation\ApiResource;
use ApiPlatform\Core\Bridge\Doctrine\Orm\Filter\OrderFilter;

/**
 * @ApiResource
 * @ApiFilter(OrderFilter::class, properties={"validFrom": { "nulls_comparison": OrderFilter::NULLS_SMALLEST, "default_direction": "DESC" }})
 */
class Offer
{
    // ...
}
```

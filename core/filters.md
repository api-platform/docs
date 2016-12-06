# Filters

API Platform Core provides a generic system to apply filters on collections. Useful filters for the Doctrine ORM are provided
with the library. You can also create custom filters that would fit your specific needs.
You can also add filtering support to your custom [data providers](data-providers.md) by implementing interfaces provided
by the library.

By default, all filters are disabled. They must be enabled explicitly.

When a filter is enabled, it is automatically documented as a `hydra:search` property in the collection response. It also
automatically appears in the [NelmioApiDoc documentation](nelmio-api-doc.md) if it is available.

## Search Filter

If Doctrine ORM support is enabled, adding filters is as easy as registering a filter service in your `app/config/services.yml`
file and adding an attribute to your resource configuration.

The search filter supports `exact`, `partial`, `start`, `end`, and `word_start` matching strategies:

* `partial` strategy uses `LIKE %text%` to search for fields that containing the text.
* `start` strategy uses `LIKE text%` to search for fields that starts with text.
* `end` strategy uses `LIKE %text` to search for fields that ends with text.
* `word_start` strategy uses `LIKE text% OR LIKE % text%` to search for fields that contains the word starting with `text`.

Prepend the letter `i` to the filter if you want it to be case insensitive. For example `ipartial` or `iexact`. Note that
this will use the `LOWER` function and **will** impact performance [if there is no proper index](performance.md#search-filter).

Case insensitivity may already be enforced at the database level depending on the [collation](https://en.wikipedia.org/wiki/Collation)
used. If you are using MySQL, note that the commonly used `utf8_unicode_ci` collation (and its sibling `utf8mb4_unicode_ci`)
are already case insensitive, as indicated by the `_ci` part in their names.

In the following example, we will see how to allow the filtering of a list of e-commerce offers:

```yaml
# app/config/services.yml

services:
    offer.search_filter:
        parent:    'api_platform.doctrine.orm.search_filter'
        arguments: [ { id: 'exact', price: 'exact', name: 'partial' } ]
        tags:      [ { name: 'api_platform.filter', id: 'offer.search' } ]
```

```php
<?php

// src/AppBundle/Entity/Offer.php

namespace AppBundle\Entity;

use ApiPlatform\Core\Annotation\ApiResource;

/**
 * @ApiResource(attributes={"filters"={"offer.search"}})
 */
class Offer
{
    // ...
}
```

`http://localhost:8000/api/offers?price=10` will return all offers with a price being exactly `10`.
`http://localhost:8000/api/offers?name=shirt` will returns all offer with a description containing the word "shirt".

Filters can be combined together: `http://localhost:8000/api/offers?price=10&name=shirt`

It is possible to filter on relations too:

```yaml
# app/config/services.yml

services:
    offer.search_filter:
        parent:    'api_platform.doctrine.orm.search_filter'
        arguments: [ { product: 'exact' } ]
        tags:      [ { name: 'api_platform.filter', id: 'offer.search' } ]
```

With this service definition, it is possible to find all offers belonging to the product identified by a given IRI.
Try the following: `http://localhost:8000/api/offers?product=/api/products/12`
Using a numeric ID is also supported: `http://localhost:8000/api/offers?product=12`

Previous URLs will return all offers for the product having the following IRI as JSON-LD identifier (`@id`): `http://localhost:8000/api/products/12`.

## Date Filter

The date filter allows to filter a collection by date intervals.

Syntax: `?property[<after|before>]=value`

The value can take any date format supported by the [`\DateTime` constructor](http://php.net/manual/en/datetime.construct.php).

As others filters, the date filter must be explicitly enabled:

```yaml
# app/config/services.yml

services:
    # Enable date filter for for the property "dateProperty" of the resource "offer"
    offer.date_filter:
        parent:    'api_platform.doctrine.orm.date_filter'
        arguments: [ { dateProperty: ~ } ]
        tags:      [ { name: 'api_platform.filter', id: 'offer.date' } ]
```

```php
<?php

// src/AppBundle/Entity/Offer.php

namespace AppBundle\Entity;

use ApiPlatform\Core\Annotation\ApiResource;

/**
 * @ApiResource(attributes={"filters"={"offer.date"}})
 */
class Offer
{
    // ...
}
```

### Managing `null` Values

The date filter is able to deal with date properties having `null` values.
Four behaviors are available at the property level of the filter:

Description                          | Strategy to set
-------------------------------------|------------------------------------------------------------------------------------
Use the default behavior of the DBMS | `null`
Exclude items                        | `ApiPlatform\Core\Bridge\Doctrine\Orm\Filter\DateFilter::EXCLUDE_NULL` (`exclude_null`)
Consider items as oldest             | `ApiPlatform\Core\Bridge\Doctrine\Orm\Filter\DateFilter::INCLUDE_NULL_BEFORE` (`include_null_before`)
Consider items as youngest           | `ApiPlatform\Core\Bridge\Doctrine\Orm\Filter\DateFilter::INCLUDE_NULL_AFTER` (`include_null_after`)

For instance, exclude entries with a property value of `null`, with the following service definition:

```yaml
# app/config/services.yml

services:
    offer.date_filter:
        parent:    'api_platform.doctrine.orm.date_filter'
        arguments: [ { dateProperty: 'exclude_null' } ]
        tags:      [ { name: 'api_platform.filter', id: 'offer.date' } ]
```

If you use a service definition format other than YAML, you can use the `ApiPlatform\Core\Bridge\Doctrine\Orm\Filter\DateFilter::EXCLUDE_NULL`
constant directly.

## Boolean Filter

The boolean filter allows you to search on boolean fields and values.

Syntax: `?property=[on|off|true|false|0|1]`

You can either use TRUE or true, the parameters are case insensitive.

Enable the filter:

```yaml
# app/config/services.yml

services:
    offer.boolean_filter:
        parent:    'api_platform.doctrine.orm.boolean_filter'
        arguments: [ { isAvailableGenericallyInMyCountry: ~ } ]
        tags:      [ { name: 'api_platform.filter', id: 'offer.boolean' } ]
```

```php
<?php

// src/AppBundle/Entity/Offer.php

namespace AppBundle\Entity;

use ApiPlatform\Core\Annotation\ApiResource;

/**
 * @ApiResource(attributes={"filters"={"offer.boolean"}})
 */
class Offer
{
    // ...
}
```

Given that the collection endpoint is `/offers`, you can filter offers by boolean  with the following query: `/offers?isAvailableGenericallyInMyCountry=true`.

It will return all offers where `isAvailableGenericallyInMyCountry` equals `true`.

## Numeric Filter

The numeric filter allows you to search on numeric fields and values.

Syntax: `?property=int|bigint|decimal...`

Enable the filter:

```yaml
# app/config/services.yml

services:
    offer.numeric_filter:
        parent:    'api_platform.doctrine.orm.numeric_filter'
        arguments: [ { sold: ~ } ]
        tags:      [ { name: 'api_platform.filter', id: 'offer.numeric' } ]
```

```php
<?php

// src/AppBundle/Entity/Offer.php

namespace AppBundle\Entity;

use ApiPlatform\Core\Annotation\ApiResource;

/**
 * @ApiResource(attributes={"filters"={"offer.numeric"}})
 */
final class Offer
{
    // ...
}
```

Given that the collection endpoint is `/offers`, you can filter offers by boolean  with the following query: `/offers?sold=1`.

It will return all offers with `sold` equals 1


## Range Filter

The range filter allows you to filter by a value Lower than, Greater than, Lower than or equal, Greater than or equal and between two values.

Syntax: `?property[lt]|[gt]|[lte]|[gte]|[between]=value`

Enable the filter:

```yaml
# app/config/services.yml

services:
    offer.numeric_filter:
        parent:    'api_platform.doctrine.orm.range_filter'
        arguments: [ { price: ~ } ]
        tags:      [ { name: 'api_platform.filter', id: 'offer.range' } ]
```

```php
<?php

// src/AppBundle/Entity/Offer.php

namespace AppBundle\Entity;

use ApiPlatform\Core\Annotation\ApiResource;

/**
 * @ApiResource(attributes={"filters"={"offer.range"}})
 */
final class Offer
{
    // ...
}
```

Given that the collection endpoint is `/offers`, you can filters the price with the following query: `/offers?price[between]=12.99..15.99`.

It will return all offers with `price` between 12.99 and 15.99.

You can filters offers by joining two value for example: `/offers?price[gt]=12.99&price[lt]=19.99`.

## Order Filter

The order filter allows to order a collection against the given properties.

Syntax: `?order[property]=<asc|desc>`

Enable the filter:

```yaml
# app/config/services.yml

services:
    offer.order_filter:
        parent:    'api_platform.doctrine.orm.order_filter'
        arguments: [ { id: ~, name: ~ } ]
        tags:      [ { name: 'api_platform.filter', id: 'offer.order' } ]
```

```php
<?php

// src/AppBundle/Entity/Offer.php

namespace AppBundle\Entity;

use ApiPlatform\Core\Annotation\ApiResource;

/**
 * @ApiResource(attributes={"filters"={"offer.order"}})
 */
class Offer
{
    // ...
}
```

Given that the collection endpoint is `/offers`, you can filter offers by name in ascending order and then by ID in descending
order with the following query: `/offers?order[name]=desc&order[id]=asc`.

By default, whenever the query does not specify the direction explicitly (e.g: `/offers?order[name]&order[id]`), filters
will not be applied unless you configure a default order direction to use:

```yaml
# app/config/services.yml

services:
    offer.order_filter:
        parent:    'api_platform.doctrine.orm.order_filter'
        arguments: [ { id: 'ASC', name: 'DESC' } ]
        tags:      [ { name: 'api_platform.filter', id: 'offer.order' } ]
```

### Using a Custom Order Query Parameter Name

A conflict will occur if `order` is also the name of a property with the search filter enabled.
Luckily, the query parameter name to use is configurable:

```yaml
# app/config/config.yml

api_platform:
    collection:
        order_parameter_name: '_order' # the URL query parameter to use is now "_order"
```

## Filtering on Nested Properties

Sometimes, you need to be able to perform filtering based on some linked resources (on the other side of a relation). All
built-in filters support nested properties using the dot (`.`) syntax, e.g.:

```yaml
# app/config/services.yml

services:
    offer.search_filter:
        parent:    'api_platform.doctrine.orm.search_filter'
        arguments: [ { product.color: 'exact' } ]
        tags:      [ { name: 'api_platform.filter', id: 'offer.search' } ]

    offer.order_filter:
        parent:    'api_platform.doctrine.orm.order_filter'
        arguments: [ { product.releaseDate: ~ } ]
        tags:      [ { name: 'api_platform.filter', id: 'offer.order' } ]
```

The above allows you to find offers by their respective product's color: `http://localhost:8000/api/offers?product.color=red`,
or order offers by the product's release date: `http://localhost:8000/api/offers?order[product.releaseDate]=desc`

## Enabling a Filter for All Properties of a Resource

As we have seen in previous examples, properties where filters can be applied must be explicitly declared. If you don't
care about security and performance (e.g. an API with restricted access), it is also possible to enable built-in filters
for all properties:

```yaml
# app/config/services.yml

services:
    # Filter enabled for all properties
    offer.order_filter:
        parent:    'api_platform.doctrine.orm.order_filter'
        arguments: [ ~ ] # This line can also be omitted
        tags:      [ { name: 'api_platform.filter', id: 'offer.order' } ]
```

**Note: Filters on nested properties must still be enabled explicitly, in order to keep things sane**

Regardless of this option, filters can by applied on a property only if:

* the property exists
* the value is supported (ex: `asc` or `desc` for the order filters).

It means that the filter will be **silently** ignored if the property:

* does not exist
* is not enabled
* has an invalid value

## Creating Custom Filters

Custom filters can be written by implementing the `ApiPlatform\Core\Api\FilterInterface`
interface.

If you use [custom data providers](data-providers.md), they must support filtering and be aware of active filters to work
properly.

### Creating Custom Doctrine ORM Filters

Doctrine ORM filters must implement the `ApiPlatform\Core\Bridge\Doctrine\Orm\Filter\FilterInterface`.
They can interact directly with the Doctrine `QueryBuilder`.

A convenient abstract class is also shipped with the bundle: `ApiPlatform\Core\Bridge\Doctrine\Orm\Filter\AbstractFilter`

### Overriding Extraction of Properties from the Request

You can change the way the filter parameters are extracted from the request. This can be done by overriding the `extractProperties(\Symfony\Component\HttpFoundation\Request $request)`
method.

In the following example, we will completely change the syntax of the order filter to be the following: `?filter[order][property]`

```php
<?php

// src/AppBundle/Filter/CustomOrderFilter.php

namespace AppBundle\Filter;

use ApiPlatform\Core\Bridge\Doctrine\Orm\Filter\OrderFilter;
use Symfony\Component\HttpFoundation\Request;

final class CustomOrderFilter extends OrderFilter
{
    protected function extractProperties(Request $request)
    {
        $filter = $request->query->get('filter[order]', []);
    }
}
```

Finally, register the custom filter:

```yaml
# app/config/services.yml

services:
    offer.custom_order_filter:
        class: 'AppBundle\Filter\CustomOrderFilter'
        tags:  [ { name: 'api_platform.filter', id: 'offer.order' } ]
```

Previous chapter: [Operations](operations.md)

Next chapter: [Serialization Groups and Relations](serialization-groups-and-relations.md)

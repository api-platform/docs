# Filters

API Platform Core provides a generic system to apply filters on collections. Useful filters for the Doctrine ORM are provided
with the library. You can also create custom filters that would fit your specific needs.
You can also add filtering support to your custom [data providers](data-providers.md) by implementing interfaces provided
by the library.

By default, all filters are disabled. They must be enabled explicitly.

When a filter is enabled, it is automatically documented as a `hydra:search` property in the collection response. It also
automatically appears in the [NelmioApiDoc documentation](nelmio-api-doc.md) if it is available.

## Doctrine ORM Filters

### Search Filter

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

### Date Filter

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

#### Managing `null` Values

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

### Boolean Filter

The boolean filter allows you to search on boolean fields and values.

Syntax: `?property=[true|false|1|0]`

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

### Numeric Filter

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


### Range Filter

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

### Order Filter

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
        arguments: [ { id: { default_direction: 'ASC' }, name: { default_direction: 'DESC' } } ]
        tags:      [ { name: 'api_platform.filter', id: 'offer.order' } ]
```

#### Comparing with Null Values

When the property used for ordering can contain `null` values, you may want to specify how `null` values are treated in
the comparison:

Description                          | Strategy to set
-------------------------------------|---------------------------------------------------------------------------------------------
Use the default behavior of the DBMS | `null`
Consider items as smallest           | `ApiPlatform\Core\Bridge\Doctrine\Orm\Filter\OrderFilter::NULLS_SMALLEST` (`nulls_smallest`)
Consider items as largest            | `ApiPlatform\Core\Bridge\Doctrine\Orm\Filter\OrderFilter::NULLS_LARGEST` (`nulls_largest`)

For instance, treat entries with a property value of `null` as the smallest, with the following service definition:

```yaml
# app/config/services.yml

services:
    offer.order_filter:
        parent:    'api_platform.doctrine.orm.date_filter'
        arguments: [ { validFrom: { nulls_comparison: 'nulls_smallest' } } ]
        tags:      [ { name: 'api_platform.filter', id: 'offer.order' } ]
```

If you use a service definition format other than YAML, you can use the `ApiPlatform\Core\Bridge\Doctrine\Orm\Filter\OrderFilter::NULLS_SMALLEST`
constant directly.

#### Using a Custom Order Query Parameter Name

A conflict will occur if `order` is also the name of a property with the search filter enabled.
Luckily, the query parameter name to use is configurable:

```yaml
# app/config/config.yml

api_platform:
    collection:
        order_parameter_name: '_order' # the URL query parameter to use is now "_order"
```

### Filtering on Nested Properties

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

### Enabling a Filter for All Properties of a Resource

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

## Serializer Filters

### Group Filter

The group filter allows you to filter by serialization groups.

Syntax: `?groups[]=<group>`

You can add as many groups as you need.

Enable the filter:

```yaml
# app/config/services.yml

services:
    book.group_filter:
        parent: 'api_platform.serializer.group_filter'
        arguments:     # Default arguments values
            - 'groups' # The query parameter name
            - false    # Allow to override the default serialization groups
        tags:   [ { name: 'api_platform.filter', id: 'book.group' } ]
```

```php
<?php

// src/AppBundle/Entity/Book.php

namespace AppBundle\Entity;

use ApiPlatform\Core\Annotation\ApiResource;

/**
 * @ApiResource(attributes={"filters"={"book.group"}})
 */
class Book
{
    // ...
}
```

Given that the collection endpoint is `/books`, you can filter by serialization groups with the following query: `/books?groups[]=read&groups[]=write`.

By default the query parameter name is `groups` but you can easily customize it to another by changing the first argument.
For example, with `serialization_groups` as name:

```yaml
# app/config/services.yml

services:
    book.group_filter:
        parent:    'api_platform.serializer.group_filter'
        arguments: [ 'serialization_groups' ]
        tags:      [ { name: 'api_platform.filter', id: 'book.group' } ]
```

You can also override the default serialization groups when you use the filter by changing the second argument to
`true`:

```yaml
# app/config/services.yml

services:
    book.group_filter:
        parent:    'api_platform.serializer.group_filter'
        arguments: [ 'groups', true ]
        tags:      [ { name: 'api_platform.filter', id: 'book.group' } ]
```

To avoid uncontrolled data exposure, you can also specify a whitelist of serialization groups as a third argument:

```yaml
# app/config/services.yml

services:
    book.group_filter:
        parent:    'api_platform.serializer.group_filter'
        arguments: [ 'groups', false, ['allowed_group', 'safe_group'] ]
        tags:      [ { name: 'api_platform.filter', id: 'book.group' } ]
```

### Property filter

The property filter add the possibility to filter serialization properties.

Syntax: `?properties[]=<property>`

You can add as many properties as you need.

Enable the filter:

```yaml
# app/config/services.yml

services:
    book.property_filter:
        parent: 'api_platform.serializer.property_filter'
        arguments:         # Default arguments values
            - 'properties' # The query parameter name
            - false        # Allow to override the default serialization properties
        tags:   [ { name: 'api_platform.filter', id: 'book.property' } ]
```

```php
<?php

// src/AppBundle/Entity/Book.php

namespace AppBundle\Entity;

use ApiPlatform\Core\Annotation\ApiResource;

/**
 * @ApiResource(attributes={"filters"={"book.property"}})
 */
class Book
{
    // ...
}
```

Given that the collection endpoint is `/books`, you can filter the serialization properties with the following query: `/books?properties[]=title&properties[]=author`.

By default the query parameter name is `properties` but you can easily customize it to another by changing the first argument.
For example, with `serialization_properties` as name:

```yaml
# app/config/services.yml

services:
    book.property_filter:
        parent:    'api_platform.serializer.property_filter'
        arguments: [ 'serialization_properties' ]
        tags:      [ { name: 'api_platform.filter', id: 'book.property' } ]
```

You can also override the default serialization properties when you use the filter by changing the second argument to
`true`:

```yaml
# app/config/services.yml

services:
    book.property_filter:
        parent:    'api_platform.serializer.property_filter'
        arguments: [ 'properties', true ]
        tags:      [ { name: 'api_platform.filter', id: 'book.property' } ]
```

To avoid uncontrolled data exposure, you can also specify a whitelist of properties as a third argument:

```yaml
# app/config/services.yml

services:
    book.group_filter:
        parent:    'api_platform.serializer.group_filter'
        arguments: [ 'groups', false, ['allowed_property', {'nested': ['safe_property']}] ]
        tags:      [ { name: 'api_platform.filter', id: 'book.group' } ]
```

## Creating Custom Filters

Custom filters can be written by implementing the `ApiPlatform\Core\Api\FilterInterface`
interface.

API Platform provides a convenient way to create Doctrine ORM filters. If you use [custom data providers](data-providers.md),
you can still create filters by implementing the previously mentioned interface, but - as API Platform isn't aware of your
persistence system's internals - you have to create the filtering logic by yourself.

### Creating Custom Doctrine ORM Filters

Doctrine filters can access to the HTTP request (Symfony's `Request` object) and to the `QueryBuilder` instance used to
retrieve data from the database. They are only applied to collections. If you want to deal with the DQL query generated
to retrieve items, or don't need to access the HTTP request, [extensions](extensions.md) are the way to go.

A Doctrine ORM filter is basically a class implementing the `ApiPlatform\Core\Bridge\Doctrine\Orm\Filter\FilterInterface`.
API Platform includes a convenient abstract class implementing this interface and providing utility methods: `ApiPlatform\Core\Bridge\Doctrine\Orm\Filter\AbstractFilter`

In the following example, we create a class to filter a collection by applying a regexp to a property. The `REGEXP` DQL
function used in this example can be found in the [`DoctrineExtensions`](https://github.com/beberlei/DoctrineExtensions)
library. This library must be properly installed and registered to use this example (works only with MySQL).

```php
<?php

// src/AppBundle/Filter/RegexpFilter.php

namespace AppBundle\Filter;

use ApiPlatform\Core\Bridge\Doctrine\Orm\Filter\AbstractFilter;
use ApiPlatform\Core\Bridge\Doctrine\Orm\Util\QueryNameGeneratorInterface;
use Doctrine\ORM\QueryBuilder;

final class RegexpFilter extends AbstractFilter
{
    protected function filterProperty(string $property, $value, QueryBuilder $queryBuilder, QueryNameGeneratorInterface $queryNameGenerator, string $resourceClass, string $operationName = null)
    {
        $parameterName = $queryNameGenerator->generateParameterName($property); // Generate a unique parameter name to avoid collisions with other filters
        $queryBuilder
            ->andWhere(sprintf('REGEXP(o.%s, :%s) = 1', $property, $parameterName))
            ->setParameter($parameterName, $value);
    }

    // This function is only used to hook in documentation generators (supported by Swagger and Hydra)
    public function getDescription(string $resourceClass): array
    {
        $description = [];
        foreach ($this->properties as $property => $strategy) {
            $description['regexp_'.$property] = [
                'property' => $property,
                'type' => 'string',
                'required' => false,
                'swagger' => ['description' => 'Filter using a regex. This will appear in the Swagger documentation!'],
            ];
        }

        return $description;
    }
}
```

Then, register this filter as a service:

```yaml
services:
    'AppBundle\Filter\RegexpFilter':
        class: 'AppBundle\Filter\RegexpFilter'
        autowire: true # See the next example for a plain old definition
        tags: [ { name: 'api_platform.filter', id: 'regexp' } ]
```

In the previous example, the filter can be applied on any property. However, thanks to the `AbstractFilter` class,
it can also be enabled for some properties:

```yaml
services:
    'AppBundle\Filter\RegexpFilter':
        class: 'AppBundle\Filter\RegexpFilter'
        arguments: [ '@doctrine', '@request_stack', '@?logger', { email: ~, anOtherProperty: ~ } ]
        tags: [ { name: 'api_platform.filter', id: 'regexp' } ]
```

Finally, add this filter to resources you want to be filtered:

```php
<?php

// src/AppBundle/Entity/Offer.php

namespace AppBundle\Entity;

use ApiPlatform\Core\Annotation\ApiResource;

/**
 * @ApiResource(attributes={"filters"={"regexp"}})
 */
class Offer
{
    // ...
}
```

You can now enable this filter using URLs like `http://example.com/offers?regexp_email=^[FOO]`. This new filter will also
appear in Swagger and Hydra documentations.

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

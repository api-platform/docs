# Filters

The bundle provides a generic system to apply filters on collections. Useful filters
for the Doctrine ORM are provided with the bundle. However the filter system is
extensible enough to let you create custom filters that would fit your specific needs
and for any data provider.

By default, all filters are disabled. They must be enabled explicitly.

When a filter is enabled, it is automatically documented as a `hydra:search` property
in collection returns. It also automatically appears in the NelmioApiDoc documentation
if this bundle is active.

## Search filter

If Doctrine ORM support is enabled, adding filters is as easy as adding an entry
in your `app/config/services.yml` file and adding a Attributes in your entity.

The search filter supports exact and partial matching strategies.
If the partial strategy is specified, an SQL query with a `WHERE` clause similar
to `LIKE %text to search%` will be automatically issued.

In the following, we will see how to allow filtering a list of e-commerce offers:

```yaml

# app/config/services.yml

services:
    offer.search_filter:
        parent:    "api_platform.doctrine.orm.search_filter"
        arguments: [ { id: "exact", price: "exact", name: "partial"  } ]
        tags:      [ { name: 'api_platform.filter', id: 'offer.search' } ]
```

# AppBundle\Entity\Offer.php
```php
<?php

namespace AppBundle\Entity;

use ApiPlatform\Core\Annotation\Property;
use ApiPlatform\Core\Annotation\Resource;
use Doctrine\ORM\Mapping as ORM;
use Symfony\Component\Validator\Constraints as Assert;

/**
 * An offer to transfer some rights to an item or to provide a service—for example, an offer to sell tickets to an event, to rent the DVD of a movie, to stream a TV show over the internet, to repair a motorcycle, or to loan a book.
 *
 * @see http://schema.org/Offer Documentation on Schema.org
 *
 * @ORM\Entity
 * @Resource(attributes={"filters"={"offer.search"}},iri="http://schema.org/Offer")
 */
class Offer
{
...
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
        parent:    "api_platform.doctrine.orm.search_filter"
        arguments: [ { id: "exact", price: "exact", name: "partial"  } ]
        tags:      [ { name: 'api_platform.filter', id: 'offer.search' } ]
```

With this service definition, it is possible to find all offers belonging to the
product identified by a given IRI.
Try the following: `http://localhost:8000/api/offers?product=/api/products/12`
Using a numeric ID is also supported: `http://localhost:8000/api/offers?product=12`

Previous URLs will return all offers for the product having the following IRI as
JSON-LD identifier (`@id`): `http://localhost:8000/api/products/12`.

## Date filter

The date filter allows to filter a collection by date intervals.

Syntax: `?property[<after|before>]=value`

The value can take any date format supported by the [`\DateTime()`](http://php.net/manual/en/datetime.construct.php)
class.

As others filters, the date filter must be explicitly enabled:

```yaml

# app/config/services.yml

services:
    # Enable date filter for for the property "dateProperty" of the resource "offer"
    offer.date_filter:
        parent:    "api_platform.doctrine.orm.date_filter"
        arguments: [ { "dateProperty": ~ } ]
        tags:      [ { name: 'api_platform.filter', id: 'offer.date' } ]
```

# AppBundle\Entity\Offer.php
```php
<?php

namespace AppBundle\Entity;

use ApiPlatform\Core\Annotation\Property;
use ApiPlatform\Core\Annotation\Resource;
use Doctrine\ORM\Mapping as ORM;
use Symfony\Component\Validator\Constraints as Assert;

/**
 * An offer to transfer some rights to an item or to provide a service—for example, an offer to sell tickets to an event, to rent the DVD of a movie, to stream a TV show over the internet, to repair a motorcycle, or to loan a book.
 *
 * @see http://schema.org/Offer Documentation on Schema.org
 *
 * @ORM\Entity
 * @Resource(attributes={"filters"={"offer.search","offer.date"}},iri="http://schema.org/Offer")
 */
class Offer
{
...
}
```

### Managing `null` values

The date filter is able to deal with date properties having `null` values.
Four behaviors are available at the property level of the filter:

| Description                          | Strategy to set                                                               |
|--------------------------------------|-------------------------------------------------------------------------------|
| Use the default behavior of the DBMS | `null`                                                                        |
| Exclude items                        | `ApiPlatform\Core\Bridge\Doctrine\Orm\Filter\DateFilter::EXCLUDE_NULL` (`0`)        |
| Consider items as oldest             | `ApiPlatform\Core\Bridge\Doctrine\Orm\Filter\DateFilter::INCLUDE_NULL_BEFORE` (`1`) |
| Consider items as youngest           | `ApiPlatform\Core\Bridge\Doctrine\Orm\Filter\DateFilter::INCLUDE_NULL_AFTER` (`2`)  |

For instance, exclude entries with a property value of `null`, with the following service definition:

```yaml

# app/config/services.yml

services:
    offer.date_filter:
        parent:    "api_platform.doctrine.orm.date_filter"
        arguments: [ { "dateProperty": ~ } ]
        tags:      [ { name: 'api_platform.filter', id: 'offer.date' }
```


If you use another service definition format than YAML, you can use the
`ApiPlatform\Core\Bridge\Doctrine\Orm\Filter\DateFilter::EXCLUDE_NULL` constant directly.

## Order filter

The order filter allows to order a collection by given properties.

Syntax: `?order[property]=<asc|desc>`

Enable the filter:

```yaml

# app/config/services.yml

services:
   offer.order_filter:
        parent:    "api_platform.doctrine.orm.order_filter"
        arguments: [ { "id": ~, "name": ~ } ]
        tags:      [ { name: 'api_platform.filter', id: 'offer.order' }
```


# AppBundle\Entity\Offer.php
```php
<?php

namespace AppBundle\Entity;

use ApiPlatform\Core\Annotation\Property;
use ApiPlatform\Core\Annotation\Resource;
use Doctrine\ORM\Mapping as ORM;
use Symfony\Component\Validator\Constraints as Assert;

/**
 * An offer to transfer some rights to an item or to provide a service—for example, an offer to sell tickets to an event, to rent the DVD of a movie, to stream a TV show over the internet, to repair a motorcycle, or to loan a book.
 *
 * @see http://schema.org/Offer Documentation on Schema.org
 *
 * @ORM\Entity
 * @Resource(attributes={"filters"={"offer.search","offer.date","offer.order"}},iri="http://schema.org/Offer")
 */
class Offer
{
...
}
```

Given that the collection endpoint is `/offers`, you can filter offers by name in
ascending order and then by ID on descending order with the following query: `/offers?order[name]=desc&order[id]=asc`.

By default, whenever the query does not specify the direction explicitly (e.g: `/offers?order[name]&order[id]`), filters will not be applied unless you configure a default order direction to use:

```yaml

# app/config/services.yml

services:
    offer.order_filter:
        parent:    "api_platform.doctrine.orm.order_filter"
        arguments: [ { "id": ASC, "name": DESC } ]
        tags:      [ { name: 'api_platform.filter', id: 'offer.order' }

    [...]
```

## Boolean Filter 

The boolean filter allow you to search on boolean fields and value.

Syntax: `?property=[on|off|true|false|0|1]`

You can either use TRUE or true, the parameters are case insensitive.

Enable the filter:

```yaml

# app/config/services.yml

services:
   offer.boolean_filter:
        parent:    "api_platform.doctrine.orm.boolean_filter"
        arguments: [ { "isAvailableGenericallyInMyCountry": ~ } ]
        tags:      [ { name: 'api_platform.filter', id: 'offer.boolean' }
```


# AppBundle\Entity\Offer.php
```php
<?php

namespace AppBundle\Entity;

use ApiPlatform\Core\Annotation\Property;
use ApiPlatform\Core\Annotation\Resource;
use Doctrine\ORM\Mapping as ORM;
use Symfony\Component\Validator\Constraints as Assert;

/**
 * An offer to transfer some rights to an item or to provide a service—for example, an offer to sell tickets to an event, to rent the DVD of a movie, to stream a TV show over the internet, to repair a motorcycle, or to loan a book.
 *
 * @see http://schema.org/Offer Documentation on Schema.org
 *
 * @ORM\Entity
 * @Resource(attributes={"filters"={"offer.search","offer.date","offer.boolean"}},iri="http://schema.org/Offer")
 */
class Offer
{
...
}
```

Given that the collection endpoint is `/offers`, you can filter offers by boolean  with the following query: `/offers?isAvailableGenericallyInMyCountry=true`.

It will return all offers with `isAvailableGenericallyInMyCountry` equals true

## Numeric Filter 

The boolean filter allow you to search on numeric fields and value.

Syntax: `?property=int|bigint|decimal...`


Enable the filter:

```yaml

# app/config/services.yml

services:
   offer.numeric_filter:
        parent:    "api_platform.doctrine.orm.numeric_filter"
        arguments: [ { "sold": ~ } ]
        tags:      [ { name: 'api_platform.filter', id: 'offer.numeric' }
```


# AppBundle\Entity\Offer.php
```php
<?php

namespace AppBundle\Entity;

use ApiPlatform\Core\Annotation\Property;
use ApiPlatform\Core\Annotation\Resource;
use Doctrine\ORM\Mapping as ORM;
use Symfony\Component\Validator\Constraints as Assert;

/**
 * An offer to transfer some rights to an item or to provide a service—for example, an offer to sell tickets to an event, to rent the DVD of a movie, to stream a TV show over the internet, to repair a motorcycle, or to loan a book.
 *
 * @see http://schema.org/Offer Documentation on Schema.org
 *
 * @ORM\Entity
 * @Resource(attributes={"filters"={"offer.search","offer.date","offer.boolean","offer.numeric"}},iri="http://schema.org/Offer")
 */
class Offer
{
...
}
```

Given that the collection endpoint is `/offers`, you can filter offers by boolean  with the following query: `/offers?sold=1`.

It will return all offers with `sold` equals 1

### Using a custom order query parameter name

A conflict will occur if `order` is also the name of a property with the search filter enabled.
Hopefully, the query parameter name to use is configurable:

```yaml

# app/config/config.yml

api_platform:
    collection:
        order_parameter_name: "_order" # the URL query parameter to use is now "_order"
```

## Filtering on nested properties

**(Added in v1.1 of the API Bundle)**

Sometimes, you need to be able to perform filtering based on some linked resources
(on the other side of a relation). All built-in filters support nested properties
using the dot (`.`) syntax, e.g.:

```yaml

# app/config/services.yml

services:
   offer.search_filter:
        parent:    "api_platform.doctrine.orm.search_filter"
        arguments: [ { "product.color": "exact" } ]
        tags:      [ { name: 'api_platform.filter', id: 'offer.searcb' }

   offer.order_filter:
        parent:    "api_platform.doctrine.orm.order_filter"
        arguments: [ { "product.releaseDate": ~ } ]
        tags:      [ { name: 'api_platform.filter', id: 'offer.order' }
```

The above allows you to find offers by their respective product's color: `http://localhost:8000/api/offers?product.color=red`,
or order offers by the product's release date: `http://localhost:8000/api/offers?order[product.releaseDate]=desc`

## Enabling a filter for all properties of a resource

As we have seen in previous examples, properties where filters can be applied must be
explicitly declared. But if you don't care about security and performance (ex:
an API with restricted access), it's also possible to enable builtin filters for
all properties:

```yaml

# app/config/services.yml

services:
    # Filter enabled for all properties
    offer.order_filter:
        parent:    "api_platform.doctrine.orm.order_filter"
        arguments: [ ~ ] # This line can also be omitted
        tags:      [ { name: 'api_platform.filter', id: 'offer.order' }
```

**Note: Filters on nested properties must still be enabled explicitly, in order to keep things sane**

Regardless of this option, filters can by applied on a property only if:
- the property exists
- the value is supported (ex: `asc` or `desc` for the order filters).

It means that the filter will be **silently** ignored if the property:
- does not exist
- is not enabled
- has an invalid value


## Creating custom filters

Custom filters can be written by implementing the `ApiPlatform\Core\Api\FilterInterface`
interface.


@TODO
What is the new way to register custom filters ?
Don't forget to register your custom filters with the `Dunglas\ApiBundle\Api\Resource::initFilters()` method.

If you use [custom data providers](data-providers.md), they must support filtering and be aware of actives filters to
work properly.

### Creating custom Doctrine ORM filters

Doctrine ORM filters must implement the `ApiPlatform\Core\Bridge\Doctrine\Orm\Filter\FilterInterface`.
They can interact directly with the Doctrine `QueryBuilder`.

A convenient abstract class is also shipped with the bundle: `ApiPlatform\Core\Bridge\Doctrine\Orm\Filter\AbstractFilter`

### Overriding extraction of properties from the request

You can change the way the filter parameters are extracted from the request. This can be done by extending the parent
filter class and overriding the `extractProperties(\Symfony\Component\HttpFoundation\Request $request)`
method.

In the following example, we will completely change the syntax of the order filter
to be the following: `?filter[order][property]`

```php

// src/AppBundle/Filter/CustomOrderFilter.php

namespace AppBundle\Filter;

use Dunglas\ApiBundle\Doctrine\Orm\OrderFilter;
use Symfony\Component\HttpFoundation\Request;

class CustomOrderFilter extends OrderFilter
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
        class:    "AppBundle\Filter\CustomOrderFilter"
        tags:      [ { name: 'api_platform.filter', id: 'offer.order' }
```

Beware: in [some cases](https://github.com/dunglas/DunglasApiBundle/issues/157#issuecomment-119576010) you may have to use double slashes in the class path to make it work:

```
services:
    offer.custom_order_filter:
        class:    "AppBundle\\Filter\\CustomOrderFilter"
        tags:      [ { name: 'api_platform.filter', id: 'offer.order' }
```

Previous chapter: [Data providers](data-providers.md)<br>
Next chapter: [Serialization groups and relations](serialization-groups-and-relations.md)

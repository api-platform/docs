# Doctrine ORM and MongoDB ODM Filters

## Introduction

For further documentation on filters (including for Eloquent and Elasticsearch), please see the [Filters documentation](filters.md).

> [!WARNING]
> For maximum flexibility and to ensure future compatibility, it is strongly recommended to configure your filters via
> the parameters attribute using `QueryParameter`. The legacy method using the `ApiFilter` attribute is not recommended.

The modern way to declare filters is to associate them directly with an operation's parameters. This allows for more
precise control over the exposed properties.

Here is the recommended approach to apply a `PartialSearchFilter` only to the title and author properties of a Book resource.

```php
<?php
// api/src/Resource/Book.php

#[ApiResource(operations: [
    new GetCollection(
        parameters: [
            // This WILL restrict to only title and author properties
            'search[:property]' => new QueryParameter(
                properties: ['title', 'author'], // Only these properties get parameters created
                filter: new PartialSearchFilter()
            )
        ]
    )
])]
class Book {
    // ...
}
```
> [!TIP]
> This filter can be also defined directly on a specific operation like `#[GetCollection(...)])` for finer
> control, like the following code:

```php
<?php
// api/src/Resource/Book.php
#[GetCollection(
    parameters: [
        // This WILL restrict to only title and author properties
        'search[:property]' => new QueryParameter(
            properties: ['title', 'author'], // Only these properties get parameters created
            filter: new PartialSearchFilter()
        )
    ]
)]
class Book {
    // ...
}
```

## Basic Knowledge

Filters are services (see the section on [custom filters](../core/filters.md#creating-custom-filters)), and they can be linked
to a Resource in two ways:

1. Through the resource declaration, as the `filters` attribute.

For example, having a filter service declaration in `services.yaml`:

```yaml
# api/config/filters.yaml
services:
  offer.date_filter:
    parent: 'api_platform.doctrine.orm.date_filter'
    arguments: [{ dateProperty: ~ }]
    tags: ['api_platform.filter']
```

> [!WARNING]
> Its discouraged to use a filter with properties in the dependency injection as it may conflict with how
> `QueryParameter` works. We recommend to use a per-parameter filter or to use the :property placeholder with a defined 
> `filterContext` specifying your strategy for a given set of parameters.

We're linking the filter `offer.date_filter` with the resource like this:

```php
<?php
// api/src/Entity/Offer.php
namespace App\Entity;

use ApiPlatform\Metadata\ApiResource;

#[GetCollection(parameters: ['date' => new QueryParameter(filter: 'offer.date_filter')])]
class Offer
{
    // ...
}
```

For MongoDB ODM, all the filters are in the namespace `ApiPlatform\Doctrine\Odm\Filter`. The filter
services all begin with `api_platform.doctrine_mongodb.odm`.

## Search Filter

> [!WARNING]
> The SearchFilter is a multi-type filter that may have inconsistencies (eg: you can search a partial date with LIKE)
> we recommend to use type-specific filters such as `PartialSearchFilter` or `DateFilter` instead.

### Built-in new Search Filters since API Platform >= 4.2

To add some search filters, choose over this new list:
- [IriFilter](#iri-filter) (filter on IRIs)
- [ExactFilter](#exact-filter) (filter with exact value)
- [PartialSearchFilter](#partial-search-filter) (filter using a `LIKE %value%`)
- [FreeTextQueryFilter](#free-text-query-filter) (allows you to apply multiple filters to multiple properties of a resource at the same time, using a single parameter in the URL)
- [OrFilter](#or-filter) (apply a filter using `orWhere` instead of `andWhere` )

### SearchFilter

If Doctrine ORM or MongoDB ODM support is enabled, using the search filter service requires you to registering a filter service in the
`api/config/services.yaml` file and adding an attribute to your resource configuration:

```yaml
    app_search_filter_via_parameter:
        parent:    'api_platform.doctrine.orm.search_filter'
        arguments: [ { 'id': 'exact', 'price': 'exact', 'description': 'partial' } ] # Declare strategies for each property
        tags:      [ { name: 'api_platform.filter', id: 'app_search_filter_via_parameter' } ]
```

The search filter supports `exact`, `partial`, `start`, `end`, and `word_start` matching strategies:

- `partial` strategy uses `LIKE %text%` to search for fields that contain `text`.
- `start` strategy uses `LIKE text%` to search for fields that start with `text`.
- `end` strategy uses `LIKE %text` to search for fields that end with `text`.
- `word_start` strategy uses `LIKE text% OR LIKE % text%` to search for fields that contain words starting with `text`.

Prepend the letter `i` to the filter if you want it to be case insensitive. For example `ipartial` or `iexact`. Note that
this will use the `LOWER` function and **will** impact performance [if there is no proper index](performance.md#search-filter).

Case insensitivity may already be enforced at the database level depending on the [collation](https://en.wikipedia.org/wiki/Collation)
used. If you are using MySQL, note that the commonly used `utf8_unicode_ci` collation (and its sibling `utf8mb4_unicode_ci`)
are already case-insensitive, as indicated by the `_ci` part in their names.

Note: Search filters with the `exact` strategy can have multiple values for the same property (in this case the condition will be similar to a SQL IN clause).

Syntax: `?property[]=foo&property[]=bar`

In the following example, we will see how to allow the filtering of e-commerce offers (a list):

```php
<?php
// api/src/Entity/Offer.php
namespace App\Entity;

use ApiPlatform\Metadata\ApiResource;
use ApiPlatform\Metadata\ApiFilter;
use ApiPlatform\Doctrine\Orm\Filter\SearchFilter;

#[GetCollection(
    parameters: [
        new QueryParameter(
            filter: 'app_search_filter_via_parameter', // the previously declared filter
            properties: ['id', 'price', 'description'],
        )
    ]
)]
class Offer
{
    // ...
}
```

`http://localhost:8000/api/offers?price=10` will return all offers with a price being exactly `10`.
`http://localhost:8000/api/offers?description=shirt` will return all offers with a description containing the word "shirt".

Filters can be combined: `http://localhost:8000/api/offers?price=10&description=shirt`

## Iri Filter

The iri filter allows filtering a resource using IRIs.

Syntax: `?property=value`

The value can take any [IRI(Internationalized Resource Identifier)](https://en.wikipedia.org/wiki/Internationalized_Resource_Identifier).

Like other [new search filters](#built-in-new-search-filters-api-platform--42) it can be used on the ApiResource attribute
or in the operation attribute, for e.g., the `#GetCollection()` attribute:

```php
// api/src/ApiResource/Chicken.php

#[GetCollection(
    parameters: [
        'chickenCoop' => new QueryParameter(filter: new IriFilter()),
    ],
)]
class Chicken
{
    //...
}
```

Given that the endpoint is `/chickens`, you can filter chickens by chicken coop with the following query:
`/chikens?chickenCoop=/chickenCoop/1`.

It will return all the chickens that live the chicken coop number 1.

## Exact Filter

The exact filter allows filtering a resource using exact values.

Syntax: `?property=value`

The value can take any scalar value or array of values.

Like other [new search filters](#built-in-new-search-filters-api-platform--42) it can be used on the ApiResource attribute
or in the operation attribute, for e.g., the `#GetCollection()` attribute:

```php
// api/src/ApiResource/Chicken.php

#[GetCollection(
    parameters: [
        'name' => new QueryParameter(filter: new ExactFilter()),
    ],
)]
class Chicken
{
    //...
}
```

Given that the endpoint is `/chickens`, you can filter chickens by name with the following query:
`/chikens?name=Gertrude`.

It will return all the chickens that are exactly named _Gertrude_.

## Partial Search Filter

The partial search filter allows filtering a resource using partial values.

Syntax: `?property=value`

The value can take any scalar value or array of values.

Like other [new search filters](#built-in-new-search-filters-api-platform--42) it can be used on the ApiResource attribute
or in the operation attribute, for e.g., the `#GetCollection()` attribute:

```php
// api/src/ApiResource/Chicken.php

#[GetCollection(
    parameters: [
        'name' => new QueryParameter(filter: new PartialSearchFilter()),
    ],
)]
class Chicken
{
    //...
}
```

Given that the endpoint is `/chickens`, you can filter chickens by name with the following query:
`/chikens?name=tom`.

It will return all chickens where the name contains the substring _tom_.

> [!NOTE]
> This filter performs a case-insensitive search. It automatically normalizes both the input value and the stored data
> (for e.g., by converting them to lowercase) before making the comparison.

## Free Text Query Filter

The free text query filter allows filtering allows you to apply a single filter across a list of properties. Its primary
role is to repeat a filter's logic for each specified field.

Syntax: `?property=value`

The value can take any scalar value or array of values.

Like other [new search filters](#built-in-new-search-filters-api-platform--42) it can be used on the ApiResource attribute
or in the operation attribute, for e.g., the `#GetCollection()` attribute:

```php
// api/src/ApiResource/Chicken.php

#[GetCollection(
    parameters: [
        'q' => new QueryParameter(
            filter: new FreeTextQueryFilter(new PartialSearchFilter()), 
            properties: ['name', 'ean']
        ),
    ],
)]
class Chicken
{
    //...
}
```

Given that the endpoint is `/chickens`, you can filter chickens by name with the following query:
`/chikens?q=tom`.

**Result**:

This request will return all chickens where:

- the `name` is exactly "FR123456"
- **AND**
- the `ean` is exactly "FR123456".

For the `OR` option refer to the [OrFilter](#or-filter).

## Or Filter

The or filter allows you to explicitly change the logical condition used by the filter it wraps. Its sole purpose is to
force a filter to combine its criteria with OR instead of the default AND.

It's the ideal tool for creating a search parameter that should find a match in any of the specified fields,
but not necessarily all of them.

Syntax: `?property=value`

The value can take any scalar value or array of values.

The `OrFilter` is a decorator: it is used by "wrapping" another, more specific filter (like for e.g. `PartialSearchFilter`
or `ExactFilter`).

The real power emerges when you combine these decorators. For instance, to create an "autocomplete" feature that finds
exact matches in one of several fields. Example of usage:

```php
// api/src/ApiResource/Chicken.php

#[GetCollection(
    parameters: [
        'autocomplete' => new QueryParameter(
            filter: new FreeTextQueryFilter(new OrFilter(new ExactFilter())), 
            properties: ['name', 'ean']
        ),
    ],
)]
class Chicken
{
    //...
}
```

Given that the endpoint is `/chickens`, you can filter chickens by name with the following query:
`/chikens?autocomplete=tom`.

**Result**:

This request will return all chickens where:

- the `name` is exactly "FR123456"
- OR
- the `ean` is exactly "FR123456".

## Date Filter

The date filter allows filtering a collection by date intervals.

Syntax: `?property[<after|before|strictly_after|strictly_before>]=value`

The value can take any date format supported by the [`\DateTime` constructor](https://www.php.net/manual/en/datetime.construct.php).

The `after` and `before` filters will filter including the value whereas `strictly_after` and `strictly_before` will filter excluding the value.

Like other filters, the Date Filter must be explicitly enabled:

### Date Filter using the QueryParameter Syntax (recommended)

```php
<?php
// api/src/ApiResource/Offer.php
namespace App\ApiResource;

#[GetCollection(
    parameters: [
        'created' => new QueryParameter(
            filter: new DateFilter(),
            property: 'createdAt' // Facultative if you use the exact property name for the parameter name (for e.g., if you use "createdAt" instead of "created", the property is auto-discovered)
        ),
    ],
)]
class Offer
{
    // ...
}
```

> [!TIP]
> For other syntaxes, for e.g., if you want to new syntax with the ApiResource attribute take a look [here](#introduction).

### Date Filter using the ApiFilter Attribute Syntax (not recommended)

Basically the ApiFilter declares the correct service under the hood. We recommend to use `QueryParameter` as they're more declarative and hide less complexity.

<code-selector>

```php
<?php
// api/src/ApiResource/Offer.php
namespace App\ApiResource;

use ApiPlatform\Metadata\ApiFilter;
use ApiPlatform\Metadata\ApiResource;
use ApiPlatform\Doctrine\Orm\Filter\DateFilter;

#[ApiResource]
#[ApiFilter(DateFilter::class, properties: ['createdAt'])]
class Offer
{
    // ...
}
```

```yaml
# config/services.yaml
services:
  offer.date_filter:
    parent: 'api_platform.doctrine.orm.date_filter'
    arguments: [{ createdAt: ~ }]
    tags: ['api_platform.filter']
    # The following are mandatory only if a _defaults section is defined with inverted values.
    # You may want to isolate filters in a dedicated file to avoid adding the following lines (by adding them in the "defaults" section)
    autowire: false
    autoconfigure: false
    public: false

# config/api/Offer.yaml
App\ApiResource\Offer:
  # ...
  operations:
    ApiPlatform\Metadata\GetCollection:
      filters: ['offer.date_filter']
```

</code-selector>

### Result using the Date Filter

Given that the collection endpoint is `/offers`, you can filter offers by date with the following query: `/offers?createdAt[after]=2018-03-19`.

It will return all offers where `createdAt` is superior or equal to `2018-03-19`.

### Managing `null` Values

The date filter is able to deal with date properties having `null` values.
Four behaviors are available at the property level of the filter:

| Description                          | Strategy to set                                                                                                           |
|--------------------------------------|---------------------------------------------------------------------------------------------------------------------------|
| Use the default behavior of the DBMS | `null`                                                                                                                    |
| Exclude items                        | `ApiPlatform\Doctrine\Common\Filter\DateFilterInterface::EXCLUDE_NULL` (`exclude_null`)                                   |
| Consider items as oldest             | `ApiPlatform\Doctrine\Common\Filter\DateFilterInterface::INCLUDE_NULL_BEFORE` (`include_null_before`)                     |
| Consider items as youngest           | `ApiPlatform\Doctrine\Common\Filter\DateFilterInterface::INCLUDE_NULL_AFTER` (`include_null_after`)                       |
| Always include items                 | `ApiPlatform\Doctrine\Common\Filter\DateFilterInterface::INCLUDE_NULL_BEFORE_AND_AFTER` (`include_null_before_and_after`) |

For instance, exclude entries with a property value of `null` with the following service definition:

#### Managing `null` Values with the Date Filter using the QueryParameter Syntax (recommended)

```php
<?php
// api/src/ApiResource/Offer.php
namespace App\ApiResource;

use ApiPlatform\Doctrine\Common\Filter\DateFilterInterface;
use ApiPlatform\Metadata\QueryParamater;
use ApiPlatform\Metadata\Get;
use ApiPlatform\Doctrine\Orm\Filter\DateFilter;

#[Get(parameters: [
    'dateProperty' => new QueryParameter(
        filter: new DateFilter(),
        filterContext: DateFilterInterface::EXCLUDE_NULL,
    ),
])]
class Offer
{
    // ...
}
```

Or you can also use the `properties` attribute on the `DateFilter` to apply your [`null` strategy](#managing-null-values):

```php
<?php
// api/src/ApiResource/Offer.php
namespace App\ApiResource;

use ApiPlatform\Doctrine\Common\Filter\DateFilterInterface;
use ApiPlatform\Metadata\QueryParamater;
use ApiPlatform\Metadata\Get;
use ApiPlatform\Doctrine\Orm\Filter\DateFilter;

#[Get(parameters: [
    'dateProperty' => new QueryParameter(
        filter: new DateFilter(properties: ['dateProperty' => DateFilterInterface::EXCLUDE_NULL]),
    ),
])]
class Offer
{
    // ...
}
```

> [!TIP]
> For other syntaxes, for e.g., if you want to new syntax with the ApiResource attribute take a look [here](#introduction).


## Boolean Filter

The boolean filter allows you to search on boolean fields and values.

Syntax: `?property=<true|false|1|0>`

Enable the filter:

### Boolean Filter using the QueryParameter Syntax

```php
<?php
// api/src/ApiResource/Offer.php
namespace App\ApiResource;

use ApiPlatform\Metadata\QueryParameter;
use ApiPlatform\Metadata\GetCollection;
use ApiPlatform\Metadata\ApiResource;
use ApiPlatform\Doctrine\Orm\Filter\BooleanFilter;

#[GetCollection(
    parameters: [
        'isAvailableGenericallyInMyCountry' => new QueryParameter(filter: new BooleanFilter()),
    ]
)]
class Offer
{
    // ...
}
```

> [!TIP]
> For other syntaxes, for e.g., if you want to new syntax with the ApiResource attribute take a look [here](#introduction).

### Result using the Boolean Filter

Given that the collection endpoint is `/offers`, you can filter offers with the following query: `/offers?isAvailableGenericallyInMyCountry=true`.

It will return all offers where `isAvailableGenericallyInMyCountry` equals `true`.

## Numeric Filter

The numeric filter allows you to search on numeric fields and values.

Syntax: `?property=<int|bigint|decimal...>`

Enable the filter:

### Numeric Filter using the QueryParameter Syntax

```php
<?php
// api/src/ApiResource/Offer.php
namespace App\ApiResource;

use ApiPlatform\Metadata\QueryParameter;
use ApiPlatform\Metadata\GetCollection;
use ApiPlatform\Metadata\ApiResource;
use ApiPlatform\Doctrine\Orm\Filter\NumericFilter;

#[GetCollection(
    parameters: [
        'sold' => new QueryParameter(filter: new NumericFilter()),
    ]
)]
class Offer
{
    // ...
}
```

> [!TIP]
> For other syntaxes, for e.g., if you want to new syntax with the ApiResource attribute take a look [here](#introduction).

### Result using the Numeric Filter

Given that the collection endpoint is `/offers`, you can filter offers with the following query: `/offers?sold=1`.

It will return all offers with `sold` equals `1`.

## Range Filter

The range filter allows you to filter by a value lower than, greater than, lower than or equal, greater than or equal and between two values.

Syntax: `?property[<lt|gt|lte|gte|between>]=value`

Enable the filter:

### Range Filter using the QueryParameter Syntax 

```php
<?php
// api/src/ApiResource/Offer.php
namespace App\ApiResource;

use ApiPlatform\Metadata\QueryParameter;
use ApiPlatform\Metadata\GetCollection;
use ApiPlatform\Metadata\ApiResource;
use ApiPlatform\Doctrine\Orm\Filter\RangeFilter;

#[GetCollection(
    parameters: [
        'price' => new QueryParameter(filter: new RangeFilter()),
    ]
)]
class Offer
{
    // ...
}
```

> [!TIP]
> For other syntaxes, for e.g., if you want to new syntax with the ApiResource attribute take a look [here](#introduction).

### Result using the Range Filter

Given that the collection endpoint is `/offers`, you can filter the price with the following query: `/offers?price[between]=12.99..15.99`.

It will return all offers with `price` between 12.99 and 15.99.

You can filter offers by joining two values, for example: `/offers?price[gt]=12.99&price[lt]=19.99`.

## Exists Filter

The "exists" filter allows you to select items based on a nullable field value.
It will also check the emptiness of a collection association.

Syntax: `?exists[property]=<true|false|1|0>`

Enable the filter:

### Exists Filter using the QueryParameter Syntax

```php
<?php
// api/src/ApiResource/Offer.php
namespace App\ApiResource;

use ApiPlatform\Metadata\QueryParameter;
use ApiPlatform\Metadata\GetCollection;
use ApiPlatform\Metadata\ApiResource;
use ApiPlatform\Doctrine\Orm\Filter\ExistsFilter;

#[GetCollection(
    parameters: [
        'transportFees' => new QueryParameter(filter: new ExistsFilter()),
    ]
)]
class Offer
{
    // ...
}
```

> [!TIP]
> For other syntaxes, for e.g., if you want to new syntax with the ApiResource attribute take a look [here](#introduction).

### Result using the Exists Filter

Given that the collection endpoint is `/offers`, you can filter offers on the nullable field with the following query: `/offers?exists[transportFees]=true`.

It will return all offers where `transportFees` is not `null`.

### Using a Custom Exists Query Parameter Name (deprecated)

> [!TIP]
> Since API Platform 4.2 defined the query parameter yourself and you don't need the above configuration.

A conflict will occur if `exists` is also the name of a property with the search filter enabled.
Luckily, the query parameter name to use is configurable:

```yaml
# api/config/packages/api_platform.yaml
api_platform:
  collection:
    exists_parameter_name: 'not_null' # the URL query parameter to use is now "not_null"
```

## Order Filter (Sorting)

The order filter allows sorting a collection against the given properties.

Syntax: `?order[property]=<asc|desc>`

Enable the filter:

### Order Filter using the QueryParameter Syntax

```php
<?php
// api/src/ApiResource/Offer.php
namespace App\ApiResource;

use ApiPlatform\Metadata\QueryParameter;
use ApiPlatform\Metadata\GetCollection;
use ApiPlatform\Metadata\ApiResource;
use ApiPlatform\Doctrine\Orm\Filter\OrderFilter;

#[GetCollection(
    parameters: [
        'transportFees' => new QueryParameter(filter: new OrderFilter()),
    ]
)]
class Offer
{
    // ...
}
```

Or you can define one Query Parameter `'order[:property]'`, which uses an Order Filter and allow you to sort on all available properties, thanks to this code:

```php
<?php
// api/src/ApiResource/Offer.php
namespace App\ApiResource;

use ApiPlatform\Doctrine\Common\Filter\OrderFilterInterface;
use ApiPlatform\Metadata\QueryParameter;
use ApiPlatform\Metadata\GetCollection;
use ApiPlatform\Metadata\ApiResource;
use ApiPlatform\Doctrine\Orm\Filter\OrderFilter;

#[GetCollection(
    parameters: [
        'order[:property]' => new QueryParameter(filter: new OrderFilter()),
    ]
)]
class Offer
{
    // ...
}
```

After that, you can use it with the following query: `/offers?order[name]=desc&order[id]=asc`.

> [!TIP]
> For other syntaxes, for e.g., if you want to new syntax with the ApiResource attribute take a look [here](#introduction).

### Result using the Order Filter

Given that the collection endpoint is `/offers`, you can filter offers by name in ascending order and then by ID in descending
order with the following query: `/offers?order[name]=desc&order[id]=asc`.

### Basic Directions Strategies with the Order Filter

By default, whenever the query does not specify the direction explicitly (e.g.: `/offers?order[name]&order[id]`), filters
will not be applied unless you configure a default order direction to use:

**Basic Strategies**

| Description | Strategy to set                                                                    |
|-------------|------------------------------------------------------------------------------------|
| Ascending   | `ApiPlatform\Doctrine\Common\Filter\OrderFilterInterface::DIRECTION_DESC` (`DESC`) |
| Descending  | `ApiPlatform\Doctrine\Common\Filter\OrderFilterInterface::DIRECTION_ASC` (`ASC`)   |

**Other Strategies**

For other sort strategies (about `null` values), please refer to the [Handling Null Values with the Order Filter section](#comparing-with-null-values-using-order-filter).

#### Order Filter Direction using the QueryParameter Syntax

```php
<?php
// api/src/ApiResource/Offer.php
namespace App\ApiResource;

use ApiPlatform\Doctrine\Common\Filter\OrderFilterInterface;
use ApiPlatform\Metadata\QueryParameter;
use ApiPlatform\Metadata\GetCollection;
use ApiPlatform\Metadata\ApiResource;
use ApiPlatform\Doctrine\Orm\Filter\OrderFilter;

#[GetCollection(
    parameters: [
        'id' => new QueryParameter(filter: new OrderFilter(), filterContext: OrderFilterInterface::DIRECTION_ASC ),
        'name' => new QueryParameter(filter: new OrderFilter(), filterContext: OrderFilterInterface::DIRECTION_DESC),
    ]
)]
class Offer
{
    // ...
}
```

Or you can also use the `properties` attribute on the `OrderFilter` to apply your [`direction` strategy](#basic-directions-strategies-with-the-order-filter):

```php
<?php
// api/src/ApiResource/Offer.php
namespace App\ApiResource;

use ApiPlatform\Doctrine\Common\Filter\OrderFilterInterface;
use ApiPlatform\Metadata\QueryParameter;
use ApiPlatform\Metadata\GetCollection;
use ApiPlatform\Metadata\ApiResource;
use ApiPlatform\Doctrine\Orm\Filter\OrderFilter;

#[GetCollection(
    parameters: [
        'id' => new QueryParameter(filter: new OrderFilter(properties: ['id' => OrderFilterInterface::DIRECTION_ASC])),
        'name' => new QueryParameter(filter: new OrderFilter(properties: ['name' => OrderFilterInterface::DIRECTION_DESC])),
    ]
)]
class Offer
{
    // ...
}
```

### Comparing with Null Values using Order Filter

When the property used for ordering can contain `null` values, you may want to specify how `null` values are treated in
the comparison:

| Description                          | Strategy to set                                                                                      |
|--------------------------------------|------------------------------------------------------------------------------------------------------|
| Use the default behavior of the DBMS | `null`                                                                                               |
| Consider items as smallest           | `ApiPlatform\Doctrine\Common\Filter\OrderFilterInterface::NULLS_SMALLEST` (`nulls_smallest`)         |
| Consider items as largest            | `ApiPlatform\Doctrine\Common\Filter\OrderFilterInterface::NULLS_LARGEST` (`nulls_largest`)           |
| Order items always first             | `ApiPlatform\Doctrine\Common\Filter\OrderFilterInterface::NULLS_ALWAYS_FIRST` (`nulls_always_first`) |
| Order items always last              | `ApiPlatform\Doctrine\Common\Filter\OrderFilterInterface::NULLS_ALWAYS_LAST` (`nulls_always_last`)   |

> [!TIP]
> For other sort strategies (including `ASC` and `DESC`), please refer to the [Handling Basic Directions with the Order Filter section](#basic-directions-strategies-with-the-order-filter).

For instance, treat entries with a property value of `null` as the smallest, with the following service definition:

### Comparing with Null Values using Order Filter using the Query Parameter Syntax

```php
<?php
// api/src/ApiResource/Offer.php
namespace App\ApiResource;

use ApiPlatform\Doctrine\Common\Filter\OrderFilterInterface;
use ApiPlatform\Metadata\QueryParameter;
use ApiPlatform\Metadata\GetCollection;
use ApiPlatform\Metadata\ApiResource;
use ApiPlatform\Doctrine\Orm\Filter\OrderFilter;

#[GetCollection(
    parameters: [
        'nulls_comparison' => new QueryParameter(
            filter: new OrderFilter(),
            property: 'validFrom',
            filterContext: OrderFilterInterface::NULLS_SMALLEST,
        ),
        'default_direction' => new QueryParameter(
            filter: new OrderFilter(),
            property: 'validFrom',
            filterContext: OrderFilterInterface::DIRECTION_DESC,
        ),
    ]
)]
class Offer
{
    // ...
}
```

## Filtering on Nested Properties

> [!WARNING]
> The legacy method using the `ApiFilter` attribute is **deprecated** and scheduled for **removal** in API Platform **5.0**.
> We strongly recommend migrating to the new `QueryParameter` syntax, which is detailed in the [Introduction](#introduction).
> For nested properties support we recommend to use a custom filter.

Sometimes, you need to be able to perform filtering based on some linked resources (on the other side of a relation). All
built-in filters support nested properties using the dot (`.`) syntax, e.g.:

<code-selector>

```php
<?php
// api/src/Entity/Offer.php
namespace App\Entity;

use ApiPlatform\Metadata\ApiFilter;
use ApiPlatform\Metadata\ApiResource;
use ApiPlatform\Doctrine\Orm\Filter\OrderFilter;
use ApiPlatform\Doctrine\Orm\Filter\SearchFilter;

#[ApiResource]
#[ApiFilter(OrderFilter::class, properties: ['product.releaseDate'])]
#[ApiFilter(SearchFilter::class, properties: ['product.color' => 'exact'])]
class Offer
{
    // ...
}
```

```yaml
# config/services.yaml
services:
  offer.order_filter:
    parent: 'api_platform.doctrine.orm.order_filter'
    arguments: [{ product.releaseDate: ~ }]
    tags: ['api_platform.filter']
    # The following are mandatory only if a _defaults section is defined with inverted values.
    # You may want to isolate filters in a dedicated file to avoid adding the following lines (by adding them in the "defaults" section)
    autowire: false
    autoconfigure: false
    public: false
  offer.search_filter:
    parent: 'api_platform.doctrine.orm.search_filter'
    arguments: [{ product.color: 'exact' }]
    tags: ['api_platform.filter']
    # The following are mandatory only if a _defaults section is defined with inverted values.
    # You may want to isolate filters in a dedicated file to avoid adding the following lines (by adding them in the "defaults" section)
    autowire: false
    autoconfigure: false
    public: false

# config/api/Offer.yaml
App\Entity\Offer:
  # ...
  operations:
    ApiPlatform\Metadata\GetCollection:
      filters: ['offer.order_filter', 'offer.search_filter']
```

</code-selector>

The above allows you to find offers by their respective product's color: `http://localhost:8000/api/offers?product.color=red`,
or order offers by the product's release date: `http://localhost:8000/api/offers?order[product.releaseDate]=desc`

## Enabling a Filter for All Properties of a Resource

> [!WARNING]
> The legacy method using the `ApiFilter` attribute is **deprecated** and scheduled for **removal** in API Platform **5.0**.
> We strongly recommend migrating to the new `QueryParameter` syntax, which is detailed in the [Introduction](#introduction).
> You can use the `:property` placeholder instead and it is recommended to use a filter for each type of data you are filtering.

As we have seen in previous examples, properties where filters can be applied must be explicitly declared. If you don't
care about security and performance (for e.g., an API with restricted access), it is also possible to enable built-in filters
for all properties:

<code-selector>

```php
<?php
// api/src/Entity/Offer.php
namespace App\Entity;

use ApiPlatform\Metadata\ApiFilter;
use ApiPlatform\Metadata\ApiResource;
use ApiPlatform\Doctrine\Orm\Filter\OrderFilter;

#[ApiResource]
#[ApiFilter(OrderFilter::class)]
class Offer
{
    // ...
}
```

```yaml
# config/services.yaml
services:
  offer.order_filter:
    parent: 'api_platform.doctrine.orm.order_filter'
    arguments: [~] # Pass null to enable the filter for all properties
    tags: ['api_platform.filter']
    # The following are mandatory only if a _defaults section is defined with inverted values.
    # You may want to isolate filters in a dedicated file to avoid adding the following lines (by adding them in the "defaults" section)
    autowire: false
    autoconfigure: false
    public: false

# config/api/Offer.yaml
App\Entity\Offer:
  # ...
  operations:
    ApiPlatform\Metadata\GetCollection:
      filters: ['offer.order_filter']
```

</code-selector>

**Note: Filters on nested properties must still be enabled explicitly to keep things sane.**

Regardless of this option, filters can be applied on a property only if:

- the property exists
- the value is supported (ex: `asc` or `desc` for the order filters).

It means that the filter will be **silently** ignored if the property:

- it does not exist
- it is not enabled
- has an invalid value


## Decorate a Doctrine filter using Symfony

A filter that implements the `ApiPlatform\Doctrine\Common\Filter\PropertyAwareFilterInterface` interface can be decorated:

```php
namespace App\Doctrine\Filter;

use ApiPlatform\Doctrine\Common\Filter\PropertyAwareFilterInterface;
use ApiPlatform\Doctrine\Orm\Filter\FilterInterface;
use ApiPlatform\Doctrine\Orm\Util\QueryNameGeneratorInterface;
use ApiPlatform\Metadata\Operation;
use Doctrine\ORM\QueryBuilder;
use Symfony\Component\DependencyInjection\Attribute\Autowire;

final class SearchTextAndDateFilter implements FilterInterface
{
    public function __construct(#[Autowire('@api_platform.doctrine.orm.search_filter.instance')] readonly FilterInterface $searchFilter, #[Autowire('@api_platform.doctrine.orm.date_filter.instance')] readonly FilterInterface $dateFilter, protected ?array $properties = null, private array $dateFilterProperties = [], private array $searchFilterProperties = [])
    {
    }

    // This function is only used to hook in documentation generators (supported by Swagger and Hydra)
    public function getDescription(string $resourceClass): array
    {
        if ($this->searchFilter instanceof PropertyAwareFilterInterface) {
            $this->searchFilter->setProperties($this->searchFilterProperties);
        }
        if ($this->dateFilter instanceof PropertyAwareFilterInterface) {
            $this->dateFilter->setProperties($this->dateFilterProperties);
        }

        return array_merge($this->searchFilter->getDescription($resourceClass), $this->dateFilter->getDescription($resourceClass));
    }

    public function apply(QueryBuilder $queryBuilder, QueryNameGeneratorInterface $queryNameGenerator, string $resourceClass, ?Operation $operation = null, array $context = []): void
    {
        if ($this->searchFilter instanceof PropertyAwareFilterInterface) {
            $this->searchFilter->setProperties($this->searchFilterProperties);
        }
        if ($this->dateFilter instanceof PropertyAwareFilterInterface) {
            $this->dateFilter->setProperties($this->dateFilterProperties);
        }

        $this->searchFilter->apply($queryBuilder, $queryNameGenerator, $resourceClass, $operation, $context);
        $this->dateFilter->apply($queryBuilder, $queryNameGenerator, $resourceClass, $operation, $context);
    }
}
```

This can be used with parameters using attributes:

```php
namespace App\Entity;

use ApiPlatform\Metadata\QueryParameter;

#[GetCollection(
    uriTemplate: 'search_filter_parameter{._format}',
    parameters: [
        'searchOnTextAndDate[:property]' => new QueryParameter(filter: 'app_filter_date_and_search'),
    ]
)]
// Note that we link the parameter filter and this filter using the "alias" option:
#[ApiFilter(SearchTextAndDateFilter::class, alias: 'app_filter_date_and_search', properties: ['foo', 'createdAt'], arguments: ['dateFilterProperties' => ['createdAt' => 'exclude_null'], 'searchFilterProperties' => ['foo' => 'exact']])]
#[ORM\Entity]
class SearchFilterParameter
{
    /**
     * @var int The id
     */
    #[ORM\Column(type: 'integer')]
    #[ORM\Id]
    #[ORM\GeneratedValue(strategy: 'AUTO')]
    private ?int $id = null;
    #[ORM\Column(type: 'string')]
    private string $foo = '';

    #[ORM\Column(type: 'datetime_immutable', nullable: true)]
    private ?\DateTimeImmutable $createdAt = null;

    public function getId(): ?int
    {
        return $this->id;
    }

    public function getFoo(): string
    {
        return $this->foo;
    }

    public function setFoo(string $foo): void
    {
        $this->foo = $foo;
    }

    public function getCreatedAt(): ?\DateTimeImmutable
    {
        return $this->createdAt;
    }

    public function setCreatedAt(\DateTimeImmutable $createdAt): void
    {
        $this->createdAt = $createdAt;
    }
}
```

## Using Doctrine ORM Filters

Doctrine ORM features [a filter system](https://www.doctrine-project.org/projects/doctrine-orm/en/current/reference/filters.html) that allows the developer to add SQL to the conditional clauses of queries, regardless of the place where the SQL is generated (for e.g., from a DQL query, or by loading associated entities).
These are applied to collections and items and therefore are incredibly useful.

The following information, specific to Doctrine filters in Symfony, is based upon [a great article posted on Michaël Perrin's blog](https://www.michaelperrin.fr/blog/2014/12/doctrine-filters).

Suppose we have a `User` entity and an `Order` entity related to the `User` one. A user should only see his orders and no one else's.

```php
<?php
// api/src/Entity/User.php
namespace App\Entity;

use ApiPlatform\Metadata\ApiResource;

#[ApiResource]
class User
{
    // ...
}
```

```php
<?php
// api/src/Entity/Order.php
namespace App\Entity;

use ApiPlatform\Metadata\ApiResource;
use Doctrine\ORM\Mapping as ORM;

#[ApiResource]
class Order
{
    // ...

    #[ORM\ManyToOne(User::class)]
    #[ORM\JoinColumn(name: 'user_id', referencedColumnName: 'id')]
    public User $user;

    // ...
}
```

The whole idea is that any query on the order table should add a `WHERE user_id = :user_id` condition.

Start by creating a custom attribute to mark restricted entities:

```php
<?php
// api/src/Attribute/UserAware.php

namespace App\Attribute;

use Attribute;

#[Attribute(Attribute::TARGET_CLASS)]
final class UserAware
{
    public $userFieldName;
}
```

Then, let's mark the `Order` entity as a "user aware" entity.

```php
<?php
// api/src/Entity/Order.php
namespace App\Entity;

use App\Attribute\UserAware;

#[UserAware(userFieldName: "user_id")]
class Order {
    // ...
}
```

Now, create a Doctrine filter class:

```php
<?php
// api/src/Filter/UserFilter.php

namespace App\Filter;

use App\Attribute\UserAware;
use Doctrine\ORM\Mapping\ClassMetadata;
use Doctrine\ORM\Query\Filter\SQLFilter;

final class UserFilter extends SQLFilter
{
    public function addFilterConstraint(ClassMetadata $targetEntity, $targetTableAlias): string
    {
        // The Doctrine filter is called for any query on any entity
        // Check if the current entity is "user aware" (marked with an attribute)
        $userAware = $targetEntity->getReflectionClass()->getAttributes(UserAware::class)[0] ?? null;

        $fieldName = $userAware?->getArguments()['userFieldName'] ?? null;
        if ($fieldName === '' || is_null($fieldName)) {
            return '';
        }

        try {
            // Don't worry, getParameter automatically escapes parameters
            $userId = $this->getParameter('id');
        } catch (\InvalidArgumentException $e) {
            // No user ID has been defined
            return '';
        }

        if (empty($fieldName) || empty($userId)) {
            return '';
        }

        return sprintf('%s.%s = %s', $targetTableAlias, $fieldName, $userId);
    }
}
```

Now, we must configure the Doctrine filter.

```yaml
# api/config/packages/api_platform.yaml
doctrine:
  orm:
    filters:
      user_filter:
        class: App\Filter\UserFilter
        enabled: true
```

Done: Doctrine will automatically filter all `UserAware`entities!

## Creating Custom Doctrine ORM Filters

Doctrine ORM filters have access to the context created from the HTTP request and to the `QueryBuilder` instance used to
retrieve data from the database. They are only applied to collections. If you want to deal with the DQL query generated
to retrieve items, [extensions](extensions.md) are the way to go.

A Doctrine ORM filter is basically a class implementing the `ApiPlatform\Doctrine\Orm\Filter\FilterInterface`.

For `MongoDB (ODM)` filters, please refer to [Creating Custom Doctrine ODM Filters documentation](#creating-custom-doctrine-mongodb-odm-filters).

### Creating Custom Doctrine ORM Filters With The New Syntax (API Platform >= 4.2)

Advantages of the new approach:

- Simplicity: No more need to extend `AbstractFilter`. A simple implementation of `FilterInterface` is all it takes.
- Clarity and Code Quality: The logic is more direct and decoupled.
- Tooling: A make command is available to generate all the boilerplate code.

#### Generating the Filter ORM Skeleton

To get started, API Platform includes a very handy make command to generate the basic structure of an ORM filter:

```console
bin/console make:filter orm
```

Then, provide the name of your filter, for example `MonthFilter`, or pass it directly as an argument:

```console
make:filter orm MyCustomFilter
```

You will get a file at `api/src/Filter/MonthFilter.php` with the following content:

```php
<?php
// api/src/Filter/MonthFilter.php

declare(strict_types=1);

namespace App\Filter;

use ApiPlatform\Doctrine\Orm\Filter\FilterInterface;
use ApiPlatform\Doctrine\Orm\Util\QueryNameGeneratorInterface;
use ApiPlatform\Metadata\BackwardCompatibleFilterDescriptionTrait;
use ApiPlatform\Metadata\Operation;
use Doctrine\ORM\QueryBuilder;

class MyCustomFilter implements FilterInterface
{
    use BackwardCompatibleFilterDescriptionTrait; // Here for backward compatibility, keep it until 5.0.

    public function apply(QueryBuilder $queryBuilder, QueryNameGeneratorInterface $queryNameGenerator, string $resourceClass, ?Operation $operation = null, array $context = []): void
    {
        // Retrieve the parameter and it's value
        // $parameter = $context['parameter'];
        // $value = $parameter->getValue();

        // Retrieve the property
        // $property = $parameter->getProperty();

        // Retrieve alias and parameter name
        // $alias = $queryBuilder->getRootAliases()[0];
        // $parameterName = $queryNameGenerator->generateParameterName($property);

        // TODO: make your awesome query using the $queryBuilder
        // $queryBuilder->
    }
}
```
#### Implementing a Custom ORM Filter

Let's create a concrete filter that allows fetching entities based on the month of a date field (for e.g., `createdAt`).

The goal is to be able to call a URL like `GET /invoices?createdAtMonth=7` to get all invoices created in July.

Here is the complete and corrected code for the filter:

```php
<?php
// api/src/Filter/MonthFilter.php

declare(strict_types=1);

namespace App\Filter;

use ApiPlatform\Doctrine\Orm\Filter\FilterInterface;
use ApiPlatform\Doctrine\Orm\Util\QueryNameGeneratorInterface;
use ApiPlatform\Metadata\BackwardCompatibleFilterDescriptionTrait;
use ApiPlatform\Metadata\Operation;
use Doctrine\ORM\QueryBuilder;

class MonthFilter implements FilterInterface
{
    use BackwardCompatibleFilterDescriptionTrait; // Here for backward compatibility, keep it until 5.0.

    public function apply(QueryBuilder $queryBuilder, QueryNameGeneratorInterface $queryNameGenerator, string $resourceClass, ?Operation $operation = null, array $context = []): void
    {
        $parameter = $context['parameter'];
        $monthValue = $parameter->getValue();

        $parameterName = $queryNameGenerator->generateParameterName($property);
        $alias = $queryBuilder->getRootAliases()[0];
        
        $queryBuilder
            ->andWhere(sprintf('MONTH(%s.%s) = :%s', $alias, $property, $parameterName))
            ->setParameter($parameterName, $monthValue);
    }
}
```

Now that the filter is created, it must be associated with an API resource. We use the `QueryParameter` object on
a `#[GetCollection]` operation attribute for this. For other syntax please refer to [this documentation](#introduction).

```php
<?php

// src/ApiResource/Invoice.php

namespace App\ApiResource;

use ApiPlatform\Metadata\QueryParameter;
use App\Filters\MonthFilter;

#[GetCollection(
    parameters: [
        'createdAtMonth' => new QueryParameter(
            filter: new MonthFilter(),
            property: 'createdAt'
        ),
    ]
)]
class Invoice
{
    // ...
}
```

And that's it! ✅

Your filter is operational.

A request like `GET /invoices?createdAtMonth=7` will now correctly return the invoices from July!

#### Adding Custom Filter ORM Validation And A Better Typing

Currently, our filter accepts any value, like `createdAtMonth=99` or `createdAtMonth=foo`, which could cause errors.
To validate inputs and ensure the correct type, we can implement the `JsonSchemaFilterInterface`.

This allows delegating validation to API Platform, respecting the [SOLID Principles](https://en.wikipedia.org/wiki/SOLID).

> [!NOTE]
> Even with our internal systems, some additional **manual validation** is needed to ensure greater accuracy. However,
> we already take care of a lot of these validations for you.
>
> You can see how this works directly in our code components:
>
> * The `ParameterValidatorProvider` for **Symfony** can be found [here](https://github.com/api-platform/core/blob/c9692b509d5b641104addbadb349b9bcab83e251/src/Symfony/Validator/State/ParameterValidatorProvider.php).
> * The `ParameterValidatorProvider` for **Laravel** is located [here](https://github.com/api-platform/core/blob/c9692b509d5b641104addbadb349b9bcab83e251/src/Laravel/State/ParameterValidatorProvider.php).
>
> Additionally, we filter out empty values within our `ParameterExtension` classes. For instance, the **Doctrine ORM**
> `ParameterExtension` [handles this filtering here](https://github.com/api-platform/core/blob/c9692b509d5b641104addbadb349b9bcab83e251/src/Doctrine/Orm/Extension/ParameterExtension.php#L51C13-L53C14).

```php
<?php

// api/src/Filters/MonthFilter.php
namespace App\Filters;

use ApiPlatform\Metadata\JsonSchemaFilterInterface;
// ...

final class MonthFilter implements FilterInterface, JsonSchemaFilterInterface
{
    public function apply(...): void {}
    
    public function getSchema(Parameter $parameter): array
    {
        return [
            'type' => 'integer',

            // <=> Symfony\Component\Validator\Constraints\Range
            'minimum' => 1,
            'maximum' => 12,
        ];
    }
}
```

With this code, under the hood, API Platform automatically adds a [Symfony Range constraint](https://symfony.com/doc/current/reference/constraints/Range.html).
This ensures the parameter only accepts values between `1` and `12` (inclusive), which is exactly what we need.

This approach offers two key benefits:

- Automatic Validation: It rejects other data types and invalid values, so you get an integer directly.
- Simplified Logic: You can retrieve the value with `$monthValue = $parameter->getValue();` knowing it's already a
- validated integer.

This means you **don't have to add custom validation to your filter class, entity, or model**. The validation is handled
for you, making your code cleaner and more efficient.

> [!TIP]
> For a complete list of constraints, see the [complete OpenApi format in the documentation](../core/filters.md#from-openapi-definition).

### Documenting the ORM Filter (OpenAPI)

#### The Simple Method (for scalar types) On A Custom ORM Filter

If your filter expects a simple type (`int`, `string`, `bool`, or arrays of these types), the quickest way is to use the
`OpenApiFilterTrait`.

```php
<?php

// api/src/Filters/MonthFilter.php
namespace App\Filters;

use ApiPlatform\Metadata\JsonSchemaFilterInterface;
use ApiPlatform\Doctrine\Common\Filter\OpenApiFilterTrait;
use ApiPlatform\Metadata\OpenApiParameterFilterInterface;
// ...

final class MonthFilter implements FilterInterface, JsonSchemaFilterInterface, OpenApiParameterFilterInterface
{
    use OpenApiFilterTrait;
    
   // ...
}
```

That's all! The trait takes care of generating the corresponding OpenAPI documentation. 🚀

#### The Custom Method to Documenting the ORM Filter (OpenAPI)

If your filter expects more complex data (an object, a specific format), you must implement the `getOpenApiParameters`
method manually.

```php
<?php

// api/src/Filter/MyComplexFilter.php

namespace App\Filter;

use ApiPlatform\OpenApi\Model\Parameter as OpenApiParameter;
use ApiPlatform\Metadata\OpenApiParameterFilterInterface;
use ApiPlatform\Metadata\Parameter;

final class MyComplexFilter implements FilterInterface, OpenApiParameterFilterInterface
{   
   public function apply(...): void {}
    
    /**
     * @return array<OpenApiParameter>
     */
    public function getOpenApiParameters(Parameter $parameter): array
    {
        // Example for a filter that expects an array of values
        // like ?myParam[key1]=value1&myParam[key2]=value2
        return [
            new OpenApiParameter(
                name: $parameter->getKey(), 
                in: 'query', 
                description: 'A custom filter for complex objects.',
                style: 'deepObject', 
                explode: true
            )
        ];
    }
}
```

## Creating Custom Doctrine MongoDB ODM Filters

For `Doctrine ORM` filters, please refer to [Creating Custom Doctrine ORM Filters documentation](#creating-custom-doctrine-orm-filters).

Doctrine MongoDB ODM filters have access to the context created from the HTTP request and to the [aggregation builder](https://www.doctrine-project.org/projects/doctrine-mongodb-odm/en/current/reference/aggregation-builder.html)
instance used to retrieve data from the database and to execute [complex operations on data](https://docs.mongodb.com/manual/aggregation/).
They are only applied to collections. If you want to deal with the aggregation pipeline generated to retrieve items, [extensions](extensions.md) are the way to go.

A Doctrine MongoDB ODM filter is basically a class implementing the `ApiPlatform\Doctrine\Odm\Filter\FilterInterface`.

### Creating Custom Doctrine ODM Filters With The New Syntax (API Platform >= 4.2)

Advantages of the new approach:

- Simplicity: No more need to extend `AbstractFilter`. A simple implementation of `FilterInterface` is all it takes.
- Clarity and Code Quality: The logic is more direct and decoupled.
- Tooling: A make command is available to generate all the boilerplate code.

#### Generating the Filter ODM Skeleton

To get started, API Platform includes a very handy make command to generate the basic structure of an ODM filter:

```console
bin/console make:filter odm
```

Then, provide the name of your filter, for example `MonthFilter`, or pass it directly as an argument:

```console
make:filter orm MyCustomFilter
```

You will get a file at `api/src/Filter/MonthFilter.php` with the following content:

```php
<?php
// api/src/Filter/MonthFilter.php

declare(strict_types=1);

namespace App\Filter;

use ApiPlatform\Doctrine\Odm\Filter\FilterInterface;
use ApiPlatform\Metadata\BackwardCompatibleFilterDescriptionTrait;
use ApiPlatform\Metadata\Operation;
use Doctrine\ODM\MongoDB\Aggregation\Builder;

class MonthFilter implements FilterInterface
{
    use BackwardCompatibleFilterDescriptionTrait; // Here for backward compatibility, keep it until 5.0.

    public function apply(Builder $aggregationBuilder, string $resourceClass, ?Operation $operation = null, array &$context = []): void
    {
        // Retrieve the parameter and it's value
        // $parameter = $context['parameter'];
        // $value = $parameter->getValue();

        // Retrieve the property
        // $property = $parameter->getProperty();

        // TODO: make your awesome query using the $aggregationBuilder
        // $aggregationBuilder->
    }
}
```
#### Implementing a Custom ODM Filter

Let's create a concrete filter that allows fetching entities based on the month of a date field (for e.g., `createdAt`).

The goal is to be able to call a URL like `GET /invoices?createdAtMonth=7` to get all invoices created in July.

Here is the complete and corrected code for the filter:

```php
<?php
// api/src/Filter/MonthFilter.php

declare(strict_types=1);

namespace App\Filter;

use ApiPlatform\Doctrine\Odm\Filter\FilterInterface;
use ApiPlatform\Metadata\BackwardCompatibleFilterDescriptionTrait;
use ApiPlatform\Metadata\Operation;
use Doctrine\ODM\MongoDB\Aggregation\Builder;

class MonthFilter implements FilterInterface
{
    use BackwardCompatibleFilterDescriptionTrait; // Here for backward compatibility, keep it until 5.0.

    public function apply(Builder $aggregationBuilder, string $resourceClass, ?Operation $operation = null, array &$context = []): void
    {
        $parameter = $context['parameter'];
        $value = $parameter->getValue();

        $property = $parameter->getProperty();

        $aggregationBuilder->match(
            $aggregationBuilder->expr()->operator('$expr', [
                '$eq' => [
                    ['$month' => '$' . $property],
                    $monthValue
                ]
            ])
        );
    }
}
```

Now that the filter is created, it must be associated with an API resource. We use the `QueryParameter` object on
a `#[GetCollection]` operation attribute for this. For other syntax please refer to [this documentation](#introduction).

```php
<?php

// src/ApiResource/Invoice.php

namespace App\ApiResource;

use ApiPlatform\Metadata\QueryParameter;
use App\Filters\MonthFilter;

#[GetCollection(
    parameters: [
        'createdAtMonth' => new QueryParameter(
            filter: new MonthFilter(), 
            property: 'createdAt'
        ),
    ]
)]
class Invoice
{
    // ...
}
```

And that's it! ✅

Your filter is operational.

A request like `GET /invoices?createdAtMonth=7` will now correctly return the invoices from July!

#### Adding Custom Filter ODM Validation And A Better Typing

Currently, our filter accepts any value, like `createdAtMonth=99` or `createdAtMonth=foo`, which could cause errors.
To validate inputs and ensure the correct type, we can implement the `JsonSchemaFilterInterface`.

This allows delegating validation to API Platform, respecting the [SOLID Principles](https://en.wikipedia.org/wiki/SOLID).

> [!NOTE]
> Even with our internal systems, some additional **manual validation** is needed to ensure greater accuracy. However,
> we already take care of a lot of these validations for you.
>
> You can see how this works directly in our code components:
>
> * The `ParameterValidatorProvider` for **Symfony** can be found [here](https://github.com/api-platform/core/blob/c9692b509d5b641104addbadb349b9bcab83e251/src/Symfony/Validator/State/ParameterValidatorProvider.php).
> * The `ParameterValidatorProvider` for **Laravel** is located [here](https://github.com/api-platform/core/blob/c9692b509d5b641104addbadb349b9bcab83e251/src/Laravel/State/ParameterValidatorProvider.php).
>
> Additionally, we filter out empty values within our `ParameterExtension` classes. For instance, the **Doctrine ODM**
> `ParameterExtension` [handles this filtering here](https://github.com/api-platform/core/blob/c9692b509d5b641104addbadb349b9bcab83e251/src/Doctrine/Odm/Extension/ParameterExtension.php#L50-L52).

```php
<?php

// api/src/Filters/MonthFilter.php
namespace App\Filters;

use ApiPlatform\Metadata\JsonSchemaFilterInterface;
// ...

final class MonthFilter implements FilterInterface, JsonSchemaFilterInterface
{
    public function apply(...): void {}
    
    public function getSchema(Parameter $parameter): array
    {
        return [
            'type' => 'integer',

            // <=> Symfony\Component\Validator\Constraints\Range
            'minimum' => 1,
            'maximum' => 12,
        ];
    }
}
```

With this code, under the hood, API Platform automatically adds a [Symfony Range constraint](https://symfony.com/doc/current/reference/constraints/Range.html).
This ensures the parameter only accepts values between `1` and `12` (inclusive), which is exactly what we need.

This approach offers two key benefits:

- Automatic Validation: It rejects other data types and invalid values, so you get an integer directly.
- Simplified Logic: You can retrieve the value with `$monthValue = $parameter->getValue();` knowing it's already a
- validated integer.

This means you **don't have to add custom validation to your filter class, entity, or model**. The validation is handled
for you, making your code cleaner and more efficient.

> [!TIP]
> For a complete list of constraints, see the [full OpenApi format in the documentation](../core/filters.md#from-openapi-definition).

### Documenting the ODM Filter (OpenAPI)

#### The Simple Method (for scalar types) On A Custom ODM Filter

If your filter expects a simple type (`int`, `string`, `bool`, or arrays of these types), the quickest way is to use the
`OpenApiFilterTrait`.

```php
<?php

// api/src/Filters/MonthFilter.php
namespace App\Filters;

use ApiPlatform\Metadata\JsonSchemaFilterInterface;
use ApiPlatform\Doctrine\Common\Filter\OpenApiFilterTrait;
use ApiPlatform\Metadata\OpenApiParameterFilterInterface;
// ...

final class MonthFilter implements FilterInterface, JsonSchemaFilterInterface, OpenApiParameterFilterInterface
{
    use OpenApiFilterTrait;
    
   // ...
}
```

That's all! The trait takes care of generating the corresponding OpenAPI documentation. 🚀

#### The Custom Method to Documenting the Filter (OpenAPI) On A Custom ODM Filter

If your filter expects more complex data (an object, a specific format), you must implement the `getOpenApiParameters`
method manually.

```php
<?php

// api/src/Filter/MyComplexFilter.php

namespace App\Filter;

use ApiPlatform\OpenApi\Model\Parameter as OpenApiParameter;
use ApiPlatform\Metadata\OpenApiParameterFilterInterface;
use ApiPlatform\Metadata\Parameter;

final class MyComplexFilter implements FilterInterface, OpenApiParameterFilterInterface
{   
   public function apply(...): void {}
    
    /**
     * @return array<OpenApiParameter>
     */
    public function getOpenApiParameters(Parameter $parameter): array
    {
        // Example for a filter that expects an array of values
        // like ?myParam[key1]=value1&myParam[key2]=value2
        return [
            new OpenApiParameter(
                name: $parameter->getKey(), 
                in: 'query', 
                description: 'A custom filter for complex objects.',
                style: 'deepObject', 
                explode: true
            )
        ];
    }
}
```

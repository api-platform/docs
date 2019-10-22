# Search Filter

The search filter supports `exact`, `partial`, `start`, `end`, and `word_start` matching strategies:

* `partial` strategy uses `LIKE %text%` to search for fields that contain `text`.
* `start` strategy uses `LIKE text%` to search for fields that start with `text`.
* `end` strategy uses `LIKE %text` to search for fields that end with `text`.
* `word_start` strategy uses `LIKE text% OR LIKE % text%` to search for fields that contain words starting with `text`.

Prepend the letter `i` to the filter if you want it to be case insensitive. For example `ipartial` or `iexact`. Note that
this will use the `LOWER` function and **will** impact performance [if there is no proper index](../caching-performance-optimization/optimizations.md#doctrine-queries-and-indexes).

Case insensitivity may already be enforced at the database level depending on the [collation](https://en.wikipedia.org/wiki/Collation)
used. If you are using MySQL, note that the commonly used `utf8_unicode_ci` collation (and its sibling `utf8mb4_unicode_ci`)
are already case-insensitive, as indicated by the `_ci` part in their names.

Note: Search filters with the `exact` strategy can have multiple values for the same property (in this case the condition will be similar to a SQL IN clause).

Syntax: `?property[]=foo&property[]=bar`

## Implementations

* Doctrine ORM: 
    * Filter class: `ApiPlatform\Core\Bridge\Doctrine\Orm\Filter\SearchFilter`
* Doctrine MongoDB ODM: 
    * Filter class: `ApiPlatform\Core\Bridge\Doctrine\MongoDbOdm\Filter\SearchFilter`
* ElasticSearch _(partial support: exact match)_:
    * Filter class: `ApiPlatform\Core\Bridge\Elasticsearch\DataProvider\Filter\TermFilter`

## Usage

In the following example, we will see how to allow the filtering of a list of e-commerce offers:

```php
<?php
// api/src/Entity/Offer.php

namespace App\Entity;

use ApiPlatform\Core\Annotation\ApiResource;
use ApiPlatform\Core\Annotation\ApiFilter;
use ApiPlatform\Core\Bridge\Doctrine\Orm\Filter\SearchFilter;

/**
 * @ApiResource()
 * @ApiFilter(SearchFilter::class, properties={"id": "exact", "price": "exact", "description": "partial"})
 */
class Offer
{
    // ...
}
```

`http://localhost:8000/api/offers?price=10` will return all offers with a price being exactly `10`.
`http://localhost:8000/api/offers?description=shirt` will return all offers with a description containing the word "shirt".

Filters can be combined together: `http://localhost:8000/api/offers?price=10&description=shirt`

It is possible to filter on relations too, if `Offer` has a `Product` relation:

```php
<?php
// api/src/Entity/Offer.php

namespace App\Entity;

use ApiPlatform\Core\Annotation\ApiResource;
use ApiPlatform\Core\Annotation\ApiFilter;
use ApiPlatform\Core\Bridge\Doctrine\Orm\Filter\SearchFilter;

/**
 * @ApiResource()
 * @ApiFilter(SearchFilter::class, properties={"product": "exact"})
 */
class Offer
{
    // ...
}
```

With this service definition, it is possible to find all offers belonging to the product identified by a given IRI.
Try the following: `http://localhost:8000/api/offers?product=/api/products/12`.
Using a numeric ID is also supported: `http://localhost:8000/api/offers?product=12`

The above URLs will return all offers for the product having the following IRI as JSON-LD identifier (`@id`): `http://localhost:8000/api/products/12`.

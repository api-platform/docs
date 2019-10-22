# Date Filter

The date filter allows to filter a collection by date intervals.

Syntax: `?property[<after|before|strictly_after|strictly_before>]=value`

The value can take any date format supported by the [`\DateTime` constructor](http://php.net/manual/en/datetime.construct.php).

The `after` and `before` filters will filter including the value whereas `strictly_after` and `strictly_before` will filter excluding the value.

## Implementations

* Doctrine ORM: 
    * Filter class: `ApiPlatform\Core\Bridge\Doctrine\Orm\Filter\DateFilter`
* Doctrine MongoDB ODM: 
    * Filter class: `ApiPlatform\Core\Bridge\Doctrine\MongoDbOdm\Filter\DateFilter`
* ElasticSearch: _Not implemented_. See [Data Providers](../fetching-and-persisting-data/data-providers.md).

## Usage

Like others filters, the date filter must be explicitly enabled:

```php
<?php
// api/src/Entity/Offer.php

namespace App\Entity;

use ApiPlatform\Core\Annotation\ApiFilter;
use ApiPlatform\Core\Annotation\ApiResource;
use ApiPlatform\Core\Bridge\Doctrine\Orm\Filter\DateFilter;

/**
 * @ApiResource
 * @ApiFilter(DateFilter::class, properties={"createdAt"})
 */
class Offer
{
    // ...
}
```

Given that the collection endpoint is `/offers`, you can filter offers by date with the following query: `/offers?createdAt[after]=2018-03-19`.

It will return all offers where `createdAt` is superior or equal to `2018-03-19`.

## Managing `null` Values

_Note: This section only applies on Doctrine ORM and Doctrine MongoDB ODM._

The date filter is able to deal with date properties having `null` values.
Four behaviors are available at the property level of the filter:

Description                          | Strategy to set
-------------------------------------|------------------------------------------------------------------------------------
Use the default behavior of the DBMS | `null`
Exclude items                        | `ApiPlatform\Core\Bridge\Doctrine\Orm\Filter\DateFilter::EXCLUDE_NULL` (`exclude_null`)
Consider items as oldest             | `ApiPlatform\Core\Bridge\Doctrine\Orm\Filter\DateFilter::INCLUDE_NULL_BEFORE` (`include_null_before`)
Consider items as youngest           | `ApiPlatform\Core\Bridge\Doctrine\Orm\Filter\DateFilter::INCLUDE_NULL_AFTER` (`include_null_after`)
Always include items                 | `ApiPlatform\Core\Bridge\Doctrine\Orm\Filter\DateFilter::INCLUDE_NULL_BEFORE_AND_AFTER` (`include_null_before_and_after`)

For instance, exclude entries with a property value of `null` with the following service definition:

```php
<?php
// api/src/Entity/Offer.php

namespace App\Entity;

use ApiPlatform\Core\Annotation\ApiFilter;
use ApiPlatform\Core\Annotation\ApiResource;
use ApiPlatform\Core\Bridge\Doctrine\Orm\Filter\DateFilter;

/**
 * @ApiResource
 * @ApiFilter(DateFilter::class, properties={"dateProperty": DateFilter::EXCLUDE_NULL})
 */
class Offer
{
    // ...
}
```

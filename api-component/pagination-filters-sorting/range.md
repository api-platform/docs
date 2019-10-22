# Range Filter

The range filter allows you to filter by a value lower than, greater than, lower than or equal, greater than or equal and between two values.

Syntax: `?property[<lt|gt|lte|gte|between>]=value`

## Implementations

* Doctrine ORM: 
    * Filter class: `ApiPlatform\Core\Bridge\Doctrine\Orm\Filter\Rangefilter`
* Doctrine MongoDB ODM: 
    * Filter class: `ApiPlatform\Core\Bridge\Doctrine\MongoDbOdm\Filter\Rangefilter`
* ElasticSearch: _Not implemented_. See [Data Providers](../fetching-and-persisting-data/data-providers.md).

## Usage

Enable the filter:

```php
<?php
// api/src/Entity/Offer.php

namespace App\Entity;

use ApiPlatform\Core\Annotation\ApiFilter;
use ApiPlatform\Core\Annotation\ApiResource;
use ApiPlatform\Core\Bridge\Doctrine\Orm\Filter\RangeFilter;

/**
 * @ApiResource
 * @ApiFilter(RangeFilter::class, properties={"price"})
 */
class Offer
{
    // ...
}
```

Given that the collection endpoint is `/offers`, you can filter the price with the following query: `/offers?price[between]=12.99..15.99`.

It will return all offers with `price` between 12.99 and 15.99.

You can filter offers by joining two values, for example: `/offers?price[gt]=12.99&price[lt]=19.99`.

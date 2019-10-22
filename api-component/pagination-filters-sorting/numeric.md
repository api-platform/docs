# Numeric Filter

The numeric filter allows you to search on numeric fields and values.

Syntax: `?property=<int|bigint|decimal...>`

## Implementations

* Doctrine ORM: 
    * Filter class: `ApiPlatform\Core\Bridge\Doctrine\Orm\Filter\NumericFilter`
* Doctrine MongoDB ODM: 
    * Filter class: `ApiPlatform\Core\Bridge\Doctrine\MongoDbOdm\Filter\NumericFilter`
* ElasticSearch: _Not implemented_. See [Data Providers](../fetching-and-persisting-data/data-providers.md).

## Usage

Enable the filter:

```php
<?php
// api/src/Entity/Offer.php

namespace App\Entity;

use ApiPlatform\Core\Annotation\ApiFilter;
use ApiPlatform\Core\Annotation\ApiResource;
use ApiPlatform\Core\Bridge\Doctrine\Orm\Filter\NumericFilter;

/**
 * @ApiResource
 * @ApiFilter(NumericFilter::class, properties={"sold"})
 */
class Offer
{
    // ...
}
```

Given that the collection endpoint is `/offers`, you can filter offers with the following query: `/offers?sold=1`.

It will return all offers with `sold` equals `1`.

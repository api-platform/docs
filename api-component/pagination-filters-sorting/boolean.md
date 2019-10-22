# Boolean Filter

The boolean filter allows you to search on boolean fields and values.

Syntax: `?property=<true|false|1|0>`

## Implementations

* Doctrine ORM: 
    * Filter class: `ApiPlatform\Core\Bridge\Doctrine\Orm\Filter\BooleanFilter`
* Doctrine MongoDB ODM: 
    * Filter class: `ApiPlatform\Core\Bridge\Doctrine\MongoDbOdm\Filter\BooleanFilter`
* ElasticSearch: _Not implemented_. See [Data Providers](../fetching-and-persisting-data/data-providers.md).

## Usage

Enable the filter:

```php
<?php
// api/src/Entity/Offer.php

namespace App\Entity;

use ApiPlatform\Core\Annotation\ApiFilter;
use ApiPlatform\Core\Annotation\ApiResource;
use ApiPlatform\Core\Bridge\Doctrine\Orm\Filter\BooleanFilter;

/**
 * @ApiResource
 * @ApiFilter(BooleanFilter::class, properties={"isAvailableGenericallyInMyCountry"})
 */
class Offer
{
    // ...
}
```

Given that the collection endpoint is `/offers`, you can filter offers with the following query: `/offers?isAvailableGenericallyInMyCountry=true`.

It will return all offers where `isAvailableGenericallyInMyCountry` equals `true`.

# Exists Filter

The exists filter allows you to select items based on a nullable field value.

Syntax: `?exists[property]=<true|false|1|0>`

Previous syntax (deprecated): `?property[exists]=<true|false|1|0>`

## Implementations

* Doctrine ORM: 
    * Filter class: `ApiPlatform\Core\Bridge\Doctrine\Orm\Filter\ExistsFilter`
* Doctrine MongoDB ODM: 
    * Filter class: `ApiPlatform\Core\Bridge\Doctrine\MongoDbOdm\Filter\ExistsFilter`
* ElasticSearch: _Not implemented_. See [Data Providers](../fetching-and-persisting-data/data-providers.md).

## Usage

Enable the filter:

```php
<?php
// api/src/Entity/Offer.php

namespace App\Entity;

use ApiPlatform\Core\Annotation\ApiFilter;
use ApiPlatform\Core\Annotation\ApiResource;
use ApiPlatform\Core\Bridge\Doctrine\Orm\Filter\ExistsFilter;

/**
 * @ApiResource
 * @ApiFilter(ExistsFilter::class, properties={"transportFees"})
 */
class Offer
{
    // ...
}
```

Given that the collection endpoint is `/offers`, you can filter offers on nullable field with the following query: `/offers?exists[transportFees]=true`.

It will return all offers where `transportFees` is not `null`.


## Using a Custom Exists Query Parameter Name

A conflict will occur if `exists` is also the name of a property with the search filter enabled.
Luckily, the query parameter name to use is configurable:

```yaml
# api/config/packages/api_platform.yaml
api_platform:
    collection:
        exists_parameter_name: 'not_null' # the URL query parameter to use is now "not_null"
```

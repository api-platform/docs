# Match Filter

The match filter allows to find resources that match the specified string against fulltext fields.

Syntax: `?property[]=value`

## Implementations

* Doctrine ORM: _Not implemented_. See [Extending API-Platform](../getting-started/extending.md).
* Doctrine MongoDB ODM: _Not implemented_. See [Extending API-Platform](../getting-started/extending.md).
* ElasticSearch ([Match Query](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-match-query.html)): 
    * Filter class: `ApiPlatform\Core\Bridge\Elasticsearch\DataProvider\Filter\MatchFilter`

## Usage

Enable the filter:

```php
<?php
// api/src/Model/Tweet.php

namespace App\Model;

use ApiPlatform\Core\Annotation\ApiFilter;
use ApiPlatform\Core\Annotation\ApiResource;
use ApiPlatform\Core\Bridge\Elasticsearch\DataProvider\Filter\MatchFilter;

/**
 * @ApiResource
 * @ApiFilter(MatchFilter::class, properties={"message"})
 */
class Tweet
{
    // ...
}
```

Given that the collection endpoint is `/tweets`, you can filter tweets by message content.

`/tweets?message=Hello%20World` will return all tweets that match the text `Hello World`.

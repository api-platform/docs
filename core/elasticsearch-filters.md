# Elasticsearch Filters

For further documentation on filters (including for Eloquent and Doctrine), please see the [Filters documentation](filters.md).

> [!WARNING]
> For maximum flexibility and to ensure future compatibility, it is strongly recommended to configure your filters via
> the parameters attribute using `QueryParameter`. The legacy method using the `ApiFilter` attribute is **deprecated** and
> will be **removed** in version **5.0**.

The modern way to declare filters is to associate them directly with an operation's parameters. This allows for more
precise control over the exposed properties.

Here is the recommended approach to apply a `MatchFilter` only to the title and author properties of a Book resource.

```php
<?php
// api/src/Resource/Book.php
#[ApiResource(operations: [
    new GetCollection(
        parameters: [
            // This WILL restrict to only title and author properties
            'search[:property]' => new QueryParameter(
                properties: ['title', 'author'], // Only these properties get parameters created
                filter: new MatchFilter()
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
            filter: new matchFilter()
        )
    ]
)]
class Book {
    // ...
}
```

**Further Reading**

- Consult the documentation on [Per-Parameter Filters (Recommended Method)](../core/filters.md#2-per-parameter-filters-recommended).
- If you are working with a legacy codebase, you can refer to the [documentation for the old syntax (deprecated)](../core/filters.md#1-legacy-filters-searchfilter-etc---not-recommended).

## Ordering Filter (Sorting)

The order filter allows to [sort](https://www.elastic.co/guide/en/elasticsearch/reference/current/search-request-sort.html)
a collection against the given properties.

Syntax: `?order[property]=<asc|desc>`

Enable the filter:

<code-selector>

```php
<?php
// api/src/Model/Tweet.php

namespace App\Model;

use ApiPlatform\Metadata\ApiFilter;
use ApiPlatform\Metadata\ApiResource;
use ApiPlatform\Elasticsearch\Filter\OrderFilter;

#[ApiResource]
#[ApiFilter(OrderFilter::class, properties: ['id', 'date'], arguments: ['orderParameterName' => 'order'])]
class Tweet
{
    // ...
}
```

```yaml
# config/services.yaml
services:
  tweet.order_filter:
    parent: 'api_platform.doctrine.orm.order_filter'
    arguments:
      $properties: { id: ~, date: ~ }
      $orderParameterName: 'order'
    tags: ['api_platform.filter']
    # The following are mandatory only if a _defaults section is defined with inverted values.
    # You may want to isolate filters in a dedicated file to avoid adding the following lines (by adding them in the defaults section)
    autowire: false
    autoconfigure: false
    public: false

# config/api/Tweet.yaml
App\Entity\Tweet:
  # ...
  filters: ['tweet.order_filter']
```

</code-selector>

Given that the collection endpoint is `/tweets`, you can filter tweets by ID and date in ascending or descending order:
`/tweets?order[id]=asc&order[date]=desc`.

By default, whenever the query does not specify the direction explicitly (e.g: `/tweets?order[id]&order[date]`), filters
will not be applied unless you configure a default order direction to use:

```php
<?php
// api/src/Model/Tweet.php

namespace App\Model;

use ApiPlatform\Metadata\ApiFilter;
use ApiPlatform\Metadata\ApiResource;
use ApiPlatform\Elasticsearch\Filter\OrderFilter;

#[ApiResource]
#[ApiFilter(OrderFilter::class, properties: ['id' => 'asc', 'date' => 'desc'])]
class Tweet
{
    // ...
}
```

### Using a Custom Order Query Parameter Name

A conflict will occur if `order` is also the name of a property with the term filter enabled. Luckily, the query
parameter name to use is configurable:

```yaml
# api/config/packages/api_platform.yaml
api_platform:
  collection:
    order_parameter_name: '_order' # the URL query parameter to use is now "_order"
```

## Match Filter

The match filter allows us to find resources that [match](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-match-query.html)
the specified text on full-text fields.

Syntax: `?property[]=value`

Enable the filter:

```php
<?php
// api/src/Model/Tweet.php

namespace App\Model;

use ApiPlatform\Metadata\ApiFilter;
use ApiPlatform\Metadata\ApiResource;
use ApiPlatform\Elasticsearch\Filter\MatchFilter;

#[ApiResource]
#[ApiFilter(MatchFilter::class, properties: ['message'])]
class Tweet
{
    // ...
}
```

Given that the collection endpoint is `/tweets`, you can filter tweets by message content.

`/tweets?message=Hello%20World` will return all tweets that match the text `Hello World`.

## Term Filter

The term filter allows us to find resources that contain the exact specified
[terms](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-term-query.html).

Syntax: `?property[]=value`

Enable the filter:

```php
<?php
// api/src/Model/User.php

namespace App\Model;

use ApiPlatform\Metadata\ApiFilter;
use ApiPlatform\Metadata\ApiResource;
use ApiPlatform\Elasticsearch\Filter\TermFilter;

#[ApiResource]
#[ApiFilter(TermFilter::class, properties: ['gender', 'age'])]
class User
{
    // ...
}
```

Given that the collection endpoint is `/users`, you can filter users by gender and age.

`/users?gender=female` will return all users whose gender is `female`.
`/users?age=42` will return all users whose age is `42`.

Filters can be combined: `/users?gender=female&age=42`.

## Filtering on Nested Properties (Elastic)

Sometimes, you need to be able to perform filtering based on some linked resources (on the other side of a relation).
All built-in filters support nested properties using the (`.`) syntax.

```php
<?php
// api/src/Model/Tweet.php

namespace App\Model;

use ApiPlatform\Metadata\ApiFilter;
use ApiPlatform\Metadata\ApiResource;
use ApiPlatform\Elasticsearch\Filter\OrderFilter;
use ApiPlatform\Elasticsearch\Filter\TermFilter;

#[ApiResource]
#[ApiFilter(OrderFilter::class, properties: ['author.firstName'])]
#[ApiFilter(TermFilter::class, properties: ['author.gender'])]
class Tweet
{
    // ...
}
```

The above allows you to find tweets by their respective author's gender `/tweets?author.gender=male`, or order tweets by the
author's first name `/tweets?order[author.firstName]=desc`.

## Creating Custom Elasticsearch Filters

Elasticsearch filters have access to the context created from the HTTP request and to the Elasticsearch query clause.
They are only applied to collections. If you want to deal with the query DSL through the search request body, extensions
are the way to go.

Existing Elasticsearch filters are applied through a [constant score query](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-constant-score-query.html).
A constant score query filter is basically a class implementing the `ApiPlatform\Elasticsearch\Filter\ConstantScoreFilterInterface`
and the `ApiPlatform\Elasticsearch\Filter\FilterInterface`. API Platform includes a convenient
abstract class implementing this last interface and providing utility methods: `ApiPlatform\Elasticsearch\Filter\AbstractFilter`.

Suppose you want to use the [match filter](#match-filter) on a property named `$fullName` and you want to add the [and operator](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-match-query.html#query-dsl-match-query-boolean) to your query:

```php
<?php
// api/src/ElasticSearch/AndOperatorFilterExtension.php

namespace App\ElasticSearch;

use ApiPlatform\Elasticsearch\Extension\RequestBodySearchCollectionExtensionInterface;
use ApiPlatform\Metadata\Operation;

class AndOperatorFilterExtension implements RequestBodySearchCollectionExtensionInterface
{
    public function applyToCollection(array $requestBody, string $resourceClass, ?Operation $operation = null, array $context = []): array;
    {
        $requestBody['query'] = $requestBody['query'] ?? [];
        $andQuery = [
            'query' => $context['filters']['fullName'],
            'operator' => 'and',
        ];

        $requestBody['query']['constant_score']['filter']['bool']['must'][0]['match']['full_name'] = $andQuery;

        return $requestBody;
    }
}
```

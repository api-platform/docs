# Custom filters

Custom filters can be written by implementing the `ApiPlatform\Core\Api\FilterInterface` interface.

API Platform provides a convenient way to create Doctrine ORM, MongoDB ODM and ElasticSearch filters. If you use [custom data providers](../fetching-and-persisting-data/data-providers.md),
you can still create filters by implementing the previously mentioned interface, but - as API Platform isn't aware of your
persistence system's internals - you have to create the filtering logic by yourself.

## Creating Custom Doctrine ORM Filters

Doctrine ORM filters have access to the context created from the HTTP request and to the `QueryBuilder` instance used to
retrieve data from the database. They are only applied to collections. If you want to deal with the DQL query generated
to retrieve items, [extensions](../fetching-and-persisting-data/extensions.md) are the way to go.

A Doctrine ORM filter is basically a class implementing the `ApiPlatform\Core\Bridge\Doctrine\Orm\Filter\FilterInterface`.
API Platform includes a convenient abstract class implementing this interface and providing utility methods: `ApiPlatform\Core\Bridge\Doctrine\Orm\Filter\AbstractFilter`.

In the following example, we create a class to filter a collection by applying a regexp to a property. The `REGEXP` DQL
function used in this example can be found in the [`DoctrineExtensions`](https://github.com/beberlei/DoctrineExtensions)
library. This library must be properly installed and registered to use this example (works only with MySQL).

```php
<?php
// api/src/Filter/RegexpFilter.php

namespace App\Filter;

use ApiPlatform\Core\Bridge\Doctrine\Orm\Filter\AbstractContextAwareFilter;
use ApiPlatform\Core\Bridge\Doctrine\Orm\Util\QueryNameGeneratorInterface;
use Doctrine\ORM\QueryBuilder;

final class RegexpFilter extends AbstractContextAwareFilter
{
    protected function filterProperty(string $property, $value, QueryBuilder $queryBuilder, QueryNameGeneratorInterface $queryNameGenerator, string $resourceClass, string $operationName = null)
    {
        // otherwise filter is applied to order and page as well
        if (
            !$this->isPropertyEnabled($property, $resourceClass) ||
            !$this->isPropertyMapped($property, $resourceClass)
        ) {
            return;
        }

        $parameterName = $queryNameGenerator->generateParameterName($property);
        $queryBuilder
            ->andWhere(sprintf('REGEXP(o.%s, :%s) = 1', $property, $parameterName))
            ->setParameter($parameterName, $value);
    }

    // This function is only used to hook in documentation generators (supported by Swagger and Hydra)
    public function getDescription(string $resourceClass): array
    {
        if (!$this->properties) {
            return [];
        }

        $description = [];
        foreach ($this->properties as $property => $strategy) {
            $description["regexp_$property"] = [
                'property' => $property,
                'type' => 'string',
                'required' => false,
                'swagger' => [
                    'description' => 'Filter using a regex. This will appear in the Swagger documentation!',
                    'name' => 'Custom name to use in the Swagger documentation',
                    'type' => 'Will appear below the name in the Swagger documentation',
                ],
            ];
        }

        return $description;
    }
}
```

Then, register this filter as a service:

```yaml
# api/config/services.yaml
services:
    # ...
    'App\Filter\RegexpFilter':
        # Uncomment only if autoconfiguration isn't enabled
        #tags: [ 'api_platform.filter' ]
```

In the previous example, the filter can be applied on any property. However, thanks to the `AbstractFilter` class,
it can also be enabled for some properties:

```yaml
# api/config/services.yaml
services:
    'App\Filter\RegexpFilter':
        arguments: [ '@doctrine', ~, '@?logger', { email: ~, anOtherProperty: ~ } ]
        # Uncomment only if autoconfiguration isn't enabled
        #tags: [ 'api_platform.filter' ]
```

Finally, add this filter to resources you want to be filtered:

```php
<?php
// api/src/Entity/Offer.php

namespace App\Entity;

use ApiPlatform\Core\Annotation\ApiResource;
use App\Filter\RegexpFilter;

/**
 * @ApiResource(attributes={"filters"={RegexpFilter::class}})
 */
class Offer
{
    // ...
}
```

Or by using the `ApiFilter` annotation:

```php
<?php
// api/src/Entity/Offer.php

namespace App\Entity;

use ApiPlatform\Core\Annotation\ApiFilter;
use ApiPlatform\Core\Annotation\ApiResource;
use App\Filter\RegexpFilter;

/**
 * @ApiResource
 * @ApiFilter(RegexpFilter::class)
 */
class Offer
{
    // ...
}
```
When using `ApiFilter` annotation, the declared properties in the `services.yaml` will not be taken into account. You have to use the `ApiFilter` way (see the [documentation](#apifilter-annotation)).

Finally you can use this filter in the URL like `http://example.com/offers?regexp_email=^[FOO]`. This new filter will also
appear in Swagger and Hydra documentations.

## Creating Custom Doctrine MongoDB ODM Filters

Doctrine MongoDB ODM filters have access to the context created from the HTTP request and to the [aggregation builder](https://www.doctrine-project.org/projects/doctrine-mongodb-odm/en/latest/reference/aggregation-builder.html)
instance used to retrieve data from the database and to execute [complex operations on data](https://docs.mongodb.com/manual/aggregation/).
They are only applied to collections. If you want to deal with the aggregation pipeline generated to retrieve items, [extensions](../fetching-and-persisting-data/extensions.md) are the way to go.

A Doctrine MongoDB ODM filter is basically a class implementing the `ApiPlatform\Core\Bridge\Doctrine\MongoDbOdm\Filter\FilterInterface`.
API Platform includes a convenient abstract class implementing this interface and providing utility methods: `ApiPlatform\Core\Bridge\Doctrine\MongoDbOdm\Filter\AbstractFilter`.

## Creating Custom Elasticsearch Filters

Elasticsearch filters have access to the context created from the HTTP request and to the Elasticsearch query clause.
They are only applied to collections. If you want to deal with the query DSL through the search request body, extensions
are the way to go.

Existing Elasticsearch filters are applied through a [constant score query](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-constant-score-query.html).
A constant score query filter is basically a class implementing the `ApiPlatform\Core\Bridge\Elasticsearch\DataProvider\Filter\ConstantScoreFilterInterface`
and the `ApiPlatform\Core\Bridge\Elasticsearch\DataProvider\Filter\FilterInterface`. API Platform includes a convenient
abstract class implementing this last interface and providing utility methods: `ApiPlatform\Core\Bridge\Elasticsearch\DataProvider\Filter\AbstractFilter`.

Suppose you want to use the [match filter](../pagination-filters-sorting/match.md) on a property named `$fullName` and you want to add the [and operator](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-match-query.html#query-dsl-match-query-boolean) to your query:

```php
<?php
// api/src/ElasticSearch/AndOperatorFilterExtension.php

namespace App\ElasticSearch;

use ApiPlatform\Core\Bridge\Elasticsearch\DataProvider\Extension\RequestBodySearchCollectionExtensionInterface;

class AndOperatorFilterExtension implements RequestBodySearchCollectionExtensionInterface
{
    public function applyToCollection(array $requestBody, string $resourceClass, ?string $operationName = null, array $context = []): array
    {
        $requestBody['query'] = $requestBody['query'] ?? [];
        $andQuery = [
            'query' => $context['pagination-filters-sorting']['fullName'],
            'operator' => 'and',
        ];
        
        $requestBody['query']['constant_score']['filter']['bool']['must'][0]['match']['full_name'] = $andQuery;
        
        return $requestBody;
    }
}
```

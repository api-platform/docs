# Performance

## Enabling the Metadata Cache

Computing metadata used by the bundle is a costly operation. Fortunately, metadata can be computed once and then cached.
API Platform internally uses a [PSR-6](http://www.php-fig.org/psr/psr-6/) cache. If the Symfony Cache Component is available
(the default in the official distribution), it automatically enables the support for the best cache adapter available.

Best performance is achieved using [APCu](https://github.com/krakjoe/apcu). Be sure to have the APCu extension installed
on your production server, API Platform will automatically use it.

## Using PPM (PHP-PM)

Response time of the API can be improved up to 15x by using [PHP Process Manager](https://github.com/php-pm/php-pm). If
you want to use it on your project, follow the documentation dedicated to Symfony on the PPM website.

Keep in mind that PPM is still in an early stage of development and can cause issues in production.

## Doctrine Queries and Indexes

### Search Filter

When using the `SearchFilter` and case insensivity, Doctrine will use the `LOWER` SQL function. Depending on your
driver, you may want to carefully index it by using a [function-based
index](http://use-the-index-luke.com/sql/where-clause/functions/case-insensitive-search) or it will impact performanc
with a huge collection. [Here are some examples to index LIKE
filters](http://use-the-index-luke.com/sql/where-clause/searching-for-ranges/like-performance-tuning) depending on your
database driver.

### Unserialized Properties Hydratation

#### Query Hint

Even though we're selecting only partial results (serialized properties) with Doctrine, it'll try to hydrate some
relations with lazy joins (for example `OneToOne` relations). It's recommended to take a look at the Symfony Profiler,
check what the generated SQL queries are doing in background and see if those may impact performance.

To force Doctrine to only hydrate partial values you need to use the
[`Query::HINT_FORCE_PARTIAL_LOAD`](http://docs.doctrine-project.org/projects/doctrine-orm/en/latest/reference/dql-doctrine-query-language.html#query-hints).
Be careful, using this query hint will force the use of partial selects. Some properties might not be available even if
you expect them. If you want to be sure that Doctrine fetches them, use eager joins and make sure that properties are
serializable.

To do this in API Platform you'd have to build a
[`QueryResultCollectionExtension`](https://github.com/api-platform/core/blob/master/src/Bridge/Doctrine/Orm/Extension/QueryResultCollectionExtensionInterface.php)
or a
[`QueryResultItemExtension`](https://github.com/api-platform/core/blob/master/src/Bridge/Doctrine/Orm/Extension/QueryResultItemExtensionInterface.php).

For example, let's decorate the existing
[`PaginationExtension`](https://github.com/api-platform/core/blob/master/src/Bridge/Doctrine/Orm/Extension/PaginationExtension.php)
by setting the query hint:

```php
<?php
// src/AppBundle/Doctrine/Orm/Extension/QueryHintPaginationExtension.php

namespace AppBundle\Doctrine\Orm\Extension;

use ApiPlatform\Core\Bridge\Doctrine\Orm\Extension\QueryResultCollectionExtensionInterface;
use ApiPlatform\Core\Bridge\Doctrine\Orm\Paginator;
use ApiPlatform\Core\Bridge\Doctrine\Orm\Util\QueryChecker;
use ApiPlatform\Core\Bridge\Doctrine\Orm\Util\QueryNameGeneratorInterface;
use Doctrine\Common\Persistence\ManagerRegistry;
use Doctrine\ORM\QueryBuilder;
use Doctrine\ORM\Tools\Pagination\Paginator as DoctrineOrmPaginator;
use Doctrine\ORM\Query;

final class QueryHintPaginationExtension implements QueryResultCollectionExtensionInterface
{
    private $managerRegistry;
    private $decorated;

    public function __construct(ManagerRegistry $managerRegistry, QueryResultCollectionExtensionInterface $decorated) {
        $this->managerRegistry = $managerRegistry;
        $this->decorated = $decorated;
    }

    /**
     * {@inheritdoc}
     */
    public function supportsResult(string $resourceClass, string $operationName = null) : bool
    {
        return $this->decorated->supportsResult($resourceClass, $operationName);
    }

    /**
     * {@inheritdoc}
     */
    public function getResult(QueryBuilder $queryBuilder)
    {
        $query = $queryBuilder->getQuery();
        // This forces doctrine to not lazy load entities
        $query->setHint(Query::HINT_FORCE_PARTIAL_LOAD, true);

        $doctrineOrmPaginator = new DoctrineOrmPaginator($query, $this->useFetchJoinCollection($queryBuilder));
        $doctrineOrmPaginator->setUseOutputWalkers($this->useOutputWalkers($queryBuilder));

        return new Paginator($doctrineOrmPaginator);
    }

    /**
     * {@inheritdoc}
     */
    public function applyToCollection(QueryBuilder $queryBuilder, QueryNameGeneratorInterface $queryNameGenerator, string $resourceClass, string $operationName = null)
    {
        return $this->decorated->applyToCollection($queryBuilder, $queryNameGenerator, $resourceClass, $operationName);
    }

    /**
     * Determines whether the Paginator should fetch join collections, if the root entity uses composite identifiers it should not.
     *
     * @see https://github.com/doctrine/doctrine2/issues/2910
     *
     * @param QueryBuilder $queryBuilder
     *
     * @return bool
     */
    private function useFetchJoinCollection(QueryBuilder $queryBuilder): bool
    {
        return !QueryChecker::hasRootEntityWithCompositeIdentifier($queryBuilder, $this->managerRegistry);
    }

    /**
     * Determines whether output walkers should be used.
     *
     * @param QueryBuilder $queryBuilder
     *
     * @return bool
     */
    private function useOutputWalkers(QueryBuilder $queryBuilder) : bool
    {
        /*
         * "Cannot count query that uses a HAVING clause. Use the output walkers for pagination"
         *
         * @see https://github.com/doctrine/doctrine2/blob/900b55d16afdcdeb5100d435a7166d3a425b9873/lib/Doctrine/ORM/Tools/Pagination/CountWalker.php#L50
         */
        if (QueryChecker::hasHavingClause($queryBuilder)) {
            return true;
        }

        /*
         * "Paginating an entity with foreign key as identifier only works when using the Output Walkers. Call Paginator#setUseOutputWalkers(true) before iterating the paginator."
         *
         * @see https://github.com/doctrine/doctrine2/blob/900b55d16afdcdeb5100d435a7166d3a425b9873/lib/Doctrine/ORM/Tools/Pagination/LimitSubqueryWalker.php#L87
         */
        if (QueryChecker::hasRootEntityWithForeignKeyIdentifier($queryBuilder, $this->managerRegistry)) {
            return true;
        }

        /*
         * "Cannot select distinct identifiers from query with LIMIT and ORDER BY on a column from a fetch joined to-many association. Use output walkers."
         *
         * @see https://github.com/doctrine/doctrine2/blob/900b55d16afdcdeb5100d435a7166d3a425b9873/lib/Doctrine/ORM/Tools/Pagination/LimitSubqueryWalker.php#L149
         */
        if (
            QueryChecker::hasMaxResults($queryBuilder) &&
            QueryChecker::hasOrderByOnToManyJoin($queryBuilder, $this->managerRegistry)
        ) {
            return true;
        }

        /*
         * When using composite identifiers pagination will need Output walkers
         */
        if (QueryChecker::hasRootEntityWithCompositeIdentifier($queryBuilder, $this->managerRegistry)) {
            return true;
        }

        // Disable output walkers by default (performance)
        return false;
    }
}
```

The service definition:

```yaml
# app/config/services.yml
services:
    app.doctrine.orm.query_extension.pagination_hint:
        class: 'AppBundle\Doctrine\Orm\Extension\QueryHintPaginationExtension'
        decorates: api_platform.doctrine.orm.query_extension.pagination
        arguments: ['@doctrine', '@api_platform.doctrine.orm.query_extension.pagination_hint.inner']
```

To alter the `Query` object on an item data provider, we can also create an `QueryHintExtension` which will alter the result:

```php
<?php
// src/AppBundle/Doctrine/Orm/Extension/QueryHintExtension.php

namespace AppBundle\Doctrine\Orm\Extension;

use ApiPlatform\Core\Bridge\Doctrine\Orm\Extension\QueryResultItemExtensionInterface;
use ApiPlatform\Core\Bridge\Doctrine\Orm\Util\QueryNameGeneratorInterface;
use Doctrine\ORM\Query;
use Doctrine\ORM\QueryBuilder;

class QueryHintExtension implements QueryResultItemExtensionInterface
{
    /**
     * {@inheritdoc}
     */
    public function applyToItem(QueryBuilder $queryBuilder, QueryNameGeneratorInterface $queryNameGenerator, string $resourceClass, array $identifiers, string $operationName = null);
    {
    }

    /**
     * {@inheritdoc}
     */
    public function supportsResult(string $resourceClass, string $operationName = null) : bool
    {
        return true;
    }

    /**
     * {@inheritdoc}
     */
    public function getResult(QueryBuilder $queryBuilder)
    {
        $query = $queryBuilder->getQuery();
        $query->setHint(Query::HINT_FORCE_PARTIAL_LOAD, true);

        return $query->getResult();
    }
}
```

The service definition:

```yaml
# app/config/services.yml

services:
    api_platform.doctrine.orm.query_extension.hint:
        class: 'AppBundle\Doctrine\Orm\Extension\QueryHintExtension'
        tags:
            - {name: api_platform.doctrine.orm.query_extension.item}
```

#### Eager loading

By default Doctrine comes with [lazy loading](http://doctrine-orm.readthedocs.io/en/latest/reference/working-with-objects.html#by-lazy-loading).
Usually a killer time-saving feature and also a performance killer with large applications.

Fortunately, Doctrine proposes another approach to remedy this problem: [eager loading](http://doctrine-orm.readthedocs.io/en/latest/reference/working-with-objects.html#by-eager-loading).
This can easily be enabled for a relation: `@ORM\ManyToOne(fetch="EAGER")`.

By default in API Platform, we made the choice to force eager loading for all relations, with or without the Doctrine
`fetch` attribute. Thanks to the eager loading [extension](extensions.md).

##### Max joins

There is a default restriction with this feature. We allow up to 30 joins per query. Beyond, an 
`ApiPlatform\Core\Exception\RuntimeException` exception will be thrown but this value can easily be increased with a
little of configuration:

```yaml
# app/config/config.yaml

api_platform:
    # ...

    eager_loading:
        max_joins: 100

    # ...
```

Be careful when you exceed this limit, it's often caused by the result of a circular reference. [Serializer groups](serialization-groups-and-relations.md)
can be a good solution to fix this issue.

##### Force eager

As mentioned above, by default we force eager loading for all relations. This behaviour can be modified with the 
configuration in order to apply it only on join relations having the `EAGER` fetch mode:

```yaml
# app/config/config.yaml

api_platform:
    # ...

    eager_loading:
        force_eager: false

    # ...
```

##### Override at resource and operation level

When eager loading is enabled, whatever the status of the `force_eager` parameter, you can easily override it directly
from the configuration of each resource. You can do this at the resource level, at the operations level, or both:

```php
<?php
// src/AppBundle/Entity/Address.php

namespace AppBundle\Entity;

use ApiPlatform\Core\Annotation\ApiResource;
use Doctrine\ORM\Mapping as ORM;

/**
 * @ApiResource
 * @ORM\Entity
 */
class Address
{
    // ...
}
```

```php
<?php
// src/AppBundle/Entity/User.php

namespace AppBundle\Entity;

use ApiPlatform\Core\Annotation\ApiResource;
use Doctrine\ORM\Mapping as ORM;

/**
 * @ApiResource(attributes={"force_eager"=false})
 * @ORM\Entity
 */
class User
{
    /**
     * @var Address
     * 
     * @ORM\ManyToOne(targetEntity="Address", fetch="EAGER") 
     */
    public $address;

    /**
     * @var Group[]
     *
     * @ORM\ManyToMany(targetEntity="Group", inversedBy="users")
     * @ORM\JoinTable(name="users_groups")
     */
    public $groups;
}
```

```php
<?php
// src/AppBundle/Entity/Group.php

namespace AppBundle\Entity;

use ApiPlatform\Core\Annotation\ApiResource;
use Doctrine\ORM\Mapping as ORM;

/**
 * @ApiResource(
 *     attributes={"force_eager"=false},
 *     itemOperations={
 *         "get"={"method"="GET", "force_eager"=true},
 *         "post"={"method"="POST"}
 *     },
 *     collectionOperations={
 *         "get"={"method"="GET", "force_eager"=true},
 *         "post"={"method"="POST"}
 *     }
 * )
 * @ORM\Entity
 */
class Group
{
    /**
     * @var User[]
     *
     * @ManyToMany(targetEntity="User", mappedBy="groups")
     */
    public $users;
}
```

Be careful, the operation level is higher priority than the resource level but both are higher priority than the global
configuration.

##### Disable eager loading

If for any reason you don't want the eager loading feature, you can turn off it in the configuration:

```yaml
# app/config/config.yaml

api_platform:
    # ...

    eager_loading:
        enabled: false

    # ...
```

The whole configuration seen before will no longer work and Doctrine will recover its default behavior.

Previous chapter: [Security](security.md)

Next chapter: [Operation Path Naming](operation-path-naming.md)

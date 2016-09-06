# Performance

## Enabling the Metadata Cache

Computing metadata used by the bundle is a costly operation. Fortunately, metadata can be computed once and then cached.
API Platform internally uses a [PSR-6](http://www.php-fig.org/psr/psr-6/) cache. If the Symfony Cache Component is available
(the default in the standard edition), it automatically enables the support for the best cache adapter available.

Best performance is achieved using [APCu](https://github.com/krakjoe/apcu). Be sure to have the APCu extension installed
on your production server, API Platform will automatically use it.

## Using PPM (PHP-PM)

Response time of the API can be improved up to 15x by using [PHP Process Manager](https://github.com/php-pm/php-pm). If
you want to use it on your project, follow the documentation dedicated to Symfony on the PPM website.

Keep in mind that PPM is still in an early stage of development and can cause issues in production.

## Doctrine Queries and Indexes

### Search Filter

When using the `SearchFilter` and case insensivity, Doctrine will be using the `LOWER` sql function. Depending on your driver, you may want to carefully index it by using a [*function-based index*](http://use-the-index-luke.com/sql/where-clause/functions/case-insensitive-search) or it will impact performances with a huge collection.

### Lazy Joins

Doctrine will do lazy joins. It's recommended to take a look at the Symfony profiler and check what SQL queries Doctrine is really doing in background. Note that `OneToOne` relations will be lazy loaded by default. To disable those in Api Platform you'd have to build a [`QueryResultExtension`](https://github.com/api-platform/core/blob/master/src/Bridge/Doctrine/Orm/Extension/QueryResultExtensionInterface.php).

For example, let's decorate the existing [`PaginationExtension`](https://github.com/api-platform/core/blob/master/src/Bridge/Doctrine/Orm/Extension/PaginationExtension.php) by setting a [Query hint](http://docs.doctrine-project.org/projects/doctrine-orm/en/latest/reference/dql-doctrine-query-language.html#query-hints).

First we can declare the decorator service:

```yaml
# app/config/services.yml

    app.doctrine.orm.query_extension.hint:
        class: 'AppBundle\Doctrine\Orm\Extension\QueryHintExtension'
        decorates: api_platform.doctrine.orm.query_extension.pagination
        arguments: ['@doctrine', '@api_platform.doctrine.orm.query_extension.hint.inner']
```

Then, our decorator would look like this:

```php
// src/AppBundle/Doctrine/Orm/Extension/QueryHintExtension.php

namespace AppBundle\Doctrine\Orm\Extension;

use ApiPlatform\Core\Bridge\Doctrine\Orm\Extension\QueryResultExtensionInterface;
use ApiPlatform\Core\Bridge\Doctrine\Orm\Paginator;
use ApiPlatform\Core\Bridge\Doctrine\Orm\Util\QueryChecker;
use ApiPlatform\Core\Bridge\Doctrine\Orm\Util\QueryNameGeneratorInterface;
use Doctrine\Common\Persistence\ManagerRegistry;
use Doctrine\ORM\QueryBuilder;
use Doctrine\ORM\Tools\Pagination\Paginator as DoctrineOrmPaginator;
use Doctrine\ORM\Query;

class QueryHintExtension implements QueryResultExtensionInterface
{
    private $managerRegistry;
    private $decorated;

    public function __construct(ManagerRegistry $managerRegistry, QueryResultExtensionInterface $decorated) {
        $this->managerRegistry = $managerRegistry;
        $this->decorated = $decorated;
    }


    /**
     * @inheritdoc
     */
    public function supportsResult(string $resourceClass, string $operationName = null) : bool
    {
        return $this->decorated->supportsResult($resourceClass, $operationName);
    }

    /**
     * @inheritdoc
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
     * @inheritdoc
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

Previous chapter: [Security](security.md)
Next chapter: [Operation Path Naming](operation-path-naming.md)

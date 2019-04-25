# Data Providers

To retrieve data exposed by the API, API Platform uses classes called **data providers**. A data provider using [Doctrine
ORM](http://www.doctrine-project.org/projects/orm.html) to retrieve data from a database, a data provider using
[Doctrine MongoDB ODM](https://www.doctrine-project.org/projects/mongodb-odm.html) to retrieve data from a document
database, and a data provider using [Elasticsearch-PHP](https://www.elastic.co/guide/en/elasticsearch/client/php-api/current/index.html)
to retrieve data from an Elasticsearch cluster are included with the library. The first one is enabled by default. These
data providers natively support paged collections and filters. They can be used as-is and are perfectly suited to common uses.

However, you sometimes want to retrieve data from other sources such as another persistence layer or a webservice.
Custom data providers can be used to do so. A project can include as many data providers as needed. The first able to
retrieve data for a given resource will be used.

For a given resource, you can implement two kinds of interface:

* the [`CollectionDataProviderInterface`](https://github.com/api-platform/core/blob/master/src/DataProvider/CollectionDataProviderInterface.php)
  is used when fetching a collection.
* the [`ItemDataProviderInterface`](https://github.com/api-platform/core/blob/master/src/DataProvider/ItemDataProviderInterface.php)
  is used when fetching items.

Both implementations can also implement a third, optional, interface called
['RestrictedDataProviderInterface'](https://github.com/api-platform/core/blob/master/src/DataProvider/RestrictedDataProviderInterface.php)
if you want to limit their effects to a single resource or operation.

In the following examples we will create custom data providers for an entity class called `App\Entity\BlogPost`.
Note, that if your entity is not Doctrine-related, you need to flag the identifier property by using `@ApiProperty(identifier=true)` for things to work properly (see also [Entity Identifier Case](serialization.md#entity-identifier-case)).

## Custom Collection Data Provider

First, your `BlogPostCollectionDataProvider` has to implement the [`CollectionDataProviderInterface`](https://github.com/api-platform/core/blob/master/src/DataProvider/CollectionDataProviderInterface.php):

The `getCollection` method must return an `array`, a `Traversable` or a [`ApiPlatform\Core\DataProvider\PaginatorInterface`](https://github.com/api-platform/core/blob/master/src/DataProvider/PaginatorInterface.php) instance.
If no data is available, you should return an empty array.

```php
<?php
// api/src/DataProvider/BlogPostCollectionDataProvider.php

namespace App\DataProvider;

use ApiPlatform\Core\DataProvider\CollectionDataProviderInterface;
use ApiPlatform\Core\DataProvider\RestrictedDataProviderInterface;
use ApiPlatform\Core\Exception\ResourceClassNotSupportedException;
use App\Entity\BlogPost;

final class BlogPostCollectionDataProvider implements CollectionDataProviderInterface, RestrictedDataProviderInterface
{
    public function supports(string $resourceClass, string $operationName = null, array $context = []): bool
    {
        return BlogPost::class === $resourceClass;
    }

    public function getCollection(string $resourceClass, string $operationName = null): \Generator
    {
        // Retrieve the blog post collection from somewhere
        yield new BlogPost(1);
        yield new BlogPost(2);
    }
}
```

If you use the default configuration, the corresponding service will be automatically registered thanks to [autowiring](https://symfony.com/doc/current/service_container/autowiring.html).
To declare the service explicitly, or to set a custom priority, you can use the following snippet:

```yaml
# api/config/services.yaml
services:
    # ...
    'App\DataProvider\BlogPostCollectionDataProvider':
        tags: [ { name: 'api_platform.collection_data_provider', priority: 2 } ]
        # Autoconfiguration must be disabled to set a custom priority
        autoconfigure: false
```

Tagging the service with the tag `api_platform.collection_data_provider` will enable API Platform Core to automatically
register and use this data provider. The optional attribute `priority` allows to define the order in which the
data providers are called. The first data provider not throwing a `ApiPlatform\Core\Exception\ResourceClassNotSupportedException`
will be used.

## Custom Item Data Provider

The process is similar for item data providers. Create a `BlogPostItemDataProvider` implementing the [`ItemDataProviderInterface`](https://github.com/api-platform/core/blob/master/src/DataProvider/ItemDataProviderInterface.php)
interface:

The `getItem` method can return `null` if no result has been found.

```php
<?php
// api/src/DataProvider/BlogPostItemDataProvider.php

namespace App\DataProvider;

use ApiPlatform\Core\DataProvider\ItemDataProviderInterface;
use ApiPlatform\Core\DataProvider\RestrictedDataProviderInterface;
use ApiPlatform\Core\Exception\ResourceClassNotSupportedException;
use App\Entity\BlogPost;

final class BlogPostItemDataProvider implements ItemDataProviderInterface, RestrictedDataProviderInterface
{
    public function supports(string $resourceClass, string $operationName = null, array $context = []): bool
    {
        return BlogPost::class === $resourceClass;
    }

    public function getItem(string $resourceClass, $id, string $operationName = null, array $context = []): ?BlogPost
    {
        // Retrieve the blog post item from somewhere then return it or null if not found
        return new BlogPost($id);
    }
}
```

If service autowiring and autoconfiguration are enabled (it's the case by default), you are done!

Otherwise, if you use a custom dependency injection configuration, you need to register the corresponding service and add the
`api_platform.item_data_provider` tag to it. As for collection data providers, the `priority` attribute can be used to order
providers.

```yaml
# api/config/services.yaml
services:
    # ...
    'App\DataProvider\BlogPostItemDataProvider': ~
        # Uncomment only if autoconfiguration is disabled
        #tags: [ 'api_platform.item_data_provider' ]
```

## Injecting the Serializer in an `ItemDataProvider`

In some cases, you may need to inject the `Serializer` in your `DataProvider`. There are no issues with the
`CollectionDataProvider`, but when injecting it in the `ItemDataProvider` it will throw a `CircularReferenceException`.

For this reason, we implemented the `SerializerAwareDataProviderInterface`:

```php
<?php
// api/src/DataProvider/BlogPostItemDataProvider.php

namespace App\DataProvider;

use ApiPlatform\Core\DataProvider\ItemDataProviderInterface;
use ApiPlatform\Core\DataProvider\SerializerAwareDataProviderInterface;
use ApiPlatform\Core\DataProvider\SerializerAwareDataProviderTrait;
use ApiPlatform\Core\Exception\ResourceClassNotSupportedException;
use App\Entity\BlogPost;

final class BlogPostItemDataProvider implements ItemDataProviderInterface, SerializerAwareDataProviderInterface
{
    use SerializerAwareDataProviderTrait;

    public function supports(string $resourceClass, string $operationName = null, array $context = []): bool
    {
        return BlogPost::class === $resourceClass;
    }

    public function getItem(string $resourceClass, $id, string $operationName = null, array $context = []): ?BlogPost
    {
        // Retrieve data from anywhere you want, in a custom format
        $data = '...';

        // Deserialize data using the Serializer
        return $this->getSerializer()->deserialize($data, BlogPost::class, 'custom');
    }
}
```

## Injecting Extensions (Pagination, Filter, EagerLoading etc.)

ApiPlatform provides a few extensions that you can reuse in your custom DataProvider.
Note that there are a few kinds of extensions which are detailed in [their own chapter of the documentation](extensions.md).
Because extensions are tagged services, you can use the [injection of tagged services](https://symfony.com/blog/new-in-symfony-3-4-simpler-injection-of-tagged-services):

```yaml
services:
    'App\DataProvider\BlogPostItemDataProvider':
        arguments:
          $itemExtensions: !tagged api_platform.doctrine.orm.query_extension.item
```

Or in XML:

```xml
<services>
    <service id="App\DataProvider\BlogPostItemDataProvider">
        <argument key="$itemExtensions" type="tagged" tag="api_platform.doctrine.orm.query_extension.item" />
    </service>
</services>
```

Your data provider will now have access to the core extensions, here is an example on how to use them:

```php
<?php
// api/src/DataProvider/BlogPostItemDataProvider.php

namespace App\DataProvider;

use ApiPlatform\Core\Bridge\Doctrine\Orm\Util\QueryNameGenerator;
use ApiPlatform\Core\DataProvider\ItemDataProviderInterface;
use ApiPlatform\Core\DataProvider\RestrictedDataProviderInterface;
use ApiPlatform\Core\Exception\ResourceClassNotSupportedException;
use App\Entity\BlogPost;
use Doctrine\Common\Persistence\ManagerRegistry;

final class BlogPostItemDataProvider implements ItemDataProviderInterface, RestrictedDataProviderInterface
{
    private $itemExtensions;
    private $managerRegistry;

    public function __construct(ManagerRegistry $managerRegistry, iterable $itemExtensions)
    {
      $this->managerRegistry = $managerRegistry;
      $this->itemExtensions = $itemExtensions;
    }

    public function supports(string $resourceClass, string $operationName = null, array $context = []): bool
    {
        return BlogPost::class === $resourceClass;
    }

    public function getItem(string $resourceClass, $id, string $operationName = null, array $context = []): ?BlogPost
    {
        $manager = $this->managerRegistry->getManagerForClass($resourceClass);
        $repository = $manager->getRepository($resourceClass);
        $queryBuilder = $repository->createQueryBuilder('o');
        $queryNameGenerator = new QueryNameGenerator();
        $identifiers = ['id' => $id];

        foreach ($this->itemExtensions as $extension) {
            $extension->applyToItem($queryBuilder, $queryNameGenerator, $resourceClass, $identifiers, $operationName, $context);
            if ($extension instanceof QueryResultItemExtensionInterface && $extension->supportsResult($resourceClass, $operationName, $context))                 {
                return $extension->getResult($queryBuilder, $resourceClass, $operationName, $context);
            }
        }

        return $queryBuilder->getQuery()->getOneOrNullResult();
    }
}

```

## Simple Array Paginator

This is an example implementation of a custom `CollectionDataProvider` that just serves an array of entities, fetched from an external API, with a custom `Paginator` to be able to use the data provider with the *API Platform Admin*.

```php
<?php
namespace App\DataProvider;

use ApiPlatform\Core\DataProvider\CollectionDataProviderInterface;
use ApiPlatform\Core\DataProvider\RestrictedDataProviderInterface;
use App\Entity\Person;
use App\Service\MyApi;
use ApiPlatform\Core\DataProvider\PaginatorInterface;
use Iterator;

final class PersonCollectionDataProvider implements CollectionDataProviderInterface, RestrictedDataProviderInterface
{
    const ITEMS_PER_PAGE = 100;

    private $api;

    public function __construct(MyApi $api)
    {
        $this->api = $api;
    }

    public function supports(string $resourceClass, string $operationName = null, array $context = []): bool
    {
        return Person::class === $resourceClass;
    }

    /**
     * @param string $resourceClass
     * @param string|null $operationName
     * @param array $context
     * @return ArrayFullPaginator
     * @throws \App\Exception\ItemNotLoadedException
     */
    public function getCollection(string $resourceClass, string $operationName = null, array $context = []): ArrayFullPaginator
    {
        $perPage = self::ITEMS_PER_PAGE;
        $page = 1;
        $api = $this->api;
        $nodes = $api->getPeopleXMLElements();

        if (isset($context['filters']['page']))
        {
            $page = (int) $context['filters']['page'];
        }

        if (isset($context['filters']['perPage']))
        {
            $perPage = (int) $context['filters']['perPage'];
        }

        $persons = [];

        // build the entity list
        foreach ($nodes as $node)
        {
            $person = $api->personFromSimpleXMLElement($node);
            $persons[] = $person;
        }

        $pagination = new ArrayFullPaginator($persons, $page, $perPage);

        return $pagination;
    }
}

/**
 * This is a paginator for collection data providers to work with items from an array,
 * that contains the full result set
 */
class ArrayFullPaginator implements Iterator, PaginatorInterface
{
    protected $position = 0;
    protected $array = [];
    protected $perPage = 100;
    protected $page = 1;

    public function __construct($items = [], $page = 1, $perPage = 30) {
        $this->array = $items;
        $this->page = $page;
        $this->perPage = $perPage;
        $this->rewind();
    }

    /**
     * Gets last page.
     */
    public function getLastPage(): float
    {
        $value = ceil($this->getTotalItems() / $this->perPage);

        return $value;
    }

    /**
     * Gets the number of items in the whole collection.
     */
    public function getTotalItems(): float
    {
        $value = $this->count();

        return $value;
    }

    /**
     * Gets the current page number.
     */
    public function getCurrentPage(): float
    {
        return $this->page;
    }

    /**
     * Gets the number of items by page.
     */
    public function getItemsPerPage(): float
    {
        return $this->perPage;
    }

    public function count() {
        $value = count($this->array);

        return $value;
    }

    public function rewind() {
        $this->position = ($this->page - 1) * $this->perPage;
    }

    public function current() {
        return $this->array[$this->position];
    }

    public function key() {
        return $this->position;
    }

    public function next() {
        ++$this->position;
    }

    public function valid() {
        $value = isset($this->array[$this->position]) &&
            ($this->position >= (($this->page - 1) * $this->perPage)) &&
            ($this->position < ($this->page * $this->perPage));

        return $value;
    }
}
```

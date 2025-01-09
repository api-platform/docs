# Data Providers

To retrieve data exposed by the API, API Platform uses classes called **data providers**. A data provider using [Doctrine
ORM](https://www.doctrine-project.org/projects/orm.html) to retrieve data from a database, a data provider using
[Doctrine MongoDB ODM](https://www.doctrine-project.org/projects/mongodb-odm.html) to retrieve data from a document
database, and a data provider using [Elasticsearch-PHP](https://www.elastic.co/guide/en/elasticsearch/client/php-api/current/index.html)
to retrieve data from an Elasticsearch cluster are included with the library. The first one is enabled by default. These
data providers natively support paged collections and filters. They can be used as-is and are perfectly suited to common uses.

However, you sometimes want to retrieve data from other sources such as another persistence layer or a webservice.
Custom data providers can be used to do so. A project can include as many data providers as needed. The first able to
retrieve data for a given resource will be used.

For a given resource, you can implement two kinds of interface:

* the [`CollectionDataProviderInterface`](https://github.com/api-platform/core/blob/2.6/src/DataProvider/CollectionDataProviderInterface.php)
  is used when fetching a collection.
* the [`ItemDataProviderInterface`](https://github.com/api-platform/core/blob/2.6/src/DataProvider/ItemDataProviderInterface.php)
  is used when fetching items.
* the [`SubresourceDataProviderInterface`](https://github.com/api-platform/core/blob/2.6/src/DataProvider/SubresourceDataProviderInterface.php)
  is used when fetching items.

Both implementations can also implement a third, optional, interface called
['RestrictedDataProviderInterface'](https://github.com/api-platform/core/blob/2.6/src/DataProvider/RestrictedDataProviderInterface.php)
if you want to limit their effects to a single resource or operation.

In the following examples we will create custom data providers for an entity class called `App\Entity\BlogPost`.
Note, that if your entity is not Doctrine-related, you need to flag the identifier property by using `#[ApiProperty(identifier: true)` for things to work properly (see also [Entity Identifier Case](serialization.md#entity-identifier-case)).

## Custom Collection Data Provider

First, your `BlogPostCollectionDataProvider` has to implement the [`CollectionDataProviderInterface`](https://github.com/api-platform/core/blob/2.6/src/DataProvider/CollectionDataProviderInterface.php):

The `getCollection` method must return an `array`, a `Traversable` or a [`ApiPlatform\Core\DataProvider\PaginatorInterface`](https://github.com/api-platform/core/blob/2.6/src/DataProvider/PaginatorInterface.php) instance.
If no data is available, you should return an empty array.

```php
<?php
// api/src/DataProvider/BlogPostCollectionDataProvider.php

namespace App\DataProvider;

use ApiPlatform\Core\DataProvider\ContextAwareCollectionDataProviderInterface;
use ApiPlatform\Core\DataProvider\RestrictedDataProviderInterface;
use App\Entity\BlogPost;

final class BlogPostCollectionDataProvider implements ContextAwareCollectionDataProviderInterface, RestrictedDataProviderInterface
{
    public function supports(string $resourceClass, string $operationName = null, array $context = []): bool
    {
        return BlogPost::class === $resourceClass;
    }

    public function getCollection(string $resourceClass, string $operationName = null, array $context = []): iterable
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
# config/services.yaml
services:
    # ...
    'App\DataProvider\BlogPostCollectionDataProvider':
        tags: [ { name: 'api_platform.collection_data_provider', priority: 2 } ]
        # Autoconfiguration must be disabled to set a custom priority
        autoconfigure: false
```

Tagging the service with the tag `api_platform.collection_data_provider` will enable API Platform Core to automatically
register and use this data provider. The optional attribute `priority` allows you to define the order in which the
data providers are called. Implementing the `ApiPlatform\Core\DataProvider\RestrictedDataProviderInterface` let you restrict the data provider use. Alternatively, you can also throw a `ApiPlatform\Core\Exception\ResourceClassNotSupportedException`. Without the `RestrictedDataProviderInterface`, the first data provider not throwing this exception will be used.

You can find a full working example in the [API Platform's demo application](https://github.com/api-platform/demo/blob/main/api/src/DataProvider/TopBookCollectionDataProvider.php).

## Custom Item Data Provider

The process is similar for item data providers. Create a `BlogPostItemDataProvider` implementing the [`ItemDataProviderInterface`](https://github.com/api-platform/core/blob/2.6/src/DataProvider/ItemDataProviderInterface.php)
interface:

The `getItem` method can return `null` if no result has been found.

```php
<?php
// api/src/DataProvider/BlogPostItemDataProvider.php

namespace App\DataProvider;

use ApiPlatform\Core\DataProvider\ItemDataProviderInterface;
use ApiPlatform\Core\DataProvider\RestrictedDataProviderInterface;
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
# config/services.yaml
services:
    # ...
    'App\DataProvider\BlogPostItemDataProvider': ~
        # Uncomment only if autoconfiguration is disabled
        #tags: [ 'api_platform.item_data_provider' ]
```

You can find a full working example in the [API Platform's demo application](https://github.com/api-platform/demo/blob/main/api/src/DataProvider/TopBookItemDataProvider.php).

## Custom Subresources Data Provider

You can add custom logic or update subresources data provider with the SubresourceDataProviderInterface .

```php
<?php
// api/src/DataProvider/BlogPostCollectionDataProvider.php

namespace App\DataProvider;

use ApiPlatform\Core\DataProvider\SubresourceDataProviderInterface;
use ApiPlatform\Core\DataProvider\RestrictedDataProviderInterface;
use App\Entity\BlogPost;

final class BlogPostSubresourceDataProvider implements SubresourceDataProviderInterface, RestrictedDataProviderInterface
{
    public function supports(string $resourceClass, string $operationName = null, array $context = []): bool
    {
        return BlogPost::class === $resourceClass;
    }

    public function getSubresource(string $resourceClass, array $identifiers, array $context, string $operationName = null): iterable
    {
        // Retrieve the blog post collection from somewhere
        $blogPosts = $this->subresourceDataProvider->getSubresource($resourceClass, $identifiers, $context, $operationName);
        // write your own logic

        return blogPosts;
    }
}
```

Declare the service in your services configuration:

```yaml
# config/services.yaml
services:
    # ...
    'App\DataProvider\BlogPostSubresourceDataProvider':
        arguments:
            - '@api_platform.doctrine.orm.default.subresource_data_provider'
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

API Platform provides a few extensions that you can reuse in your custom DataProvider.
Note that there are a few kinds of extensions which are detailed in [their own chapter of the documentation](extensions.md).
Because extensions are tagged services, you can use the [injection of tagged services](https://symfony.com/blog/new-in-symfony-3-4-simpler-injection-of-tagged-services):

<code-selector>

```yaml
services:
    'App\DataProvider\BlogPostItemDataProvider':
        arguments:
          $itemExtensions: !tagged api_platform.doctrine.orm.query_extension.item
```

```xml
<services>
    <service id="App\DataProvider\BlogPostItemDataProvider">
        <argument key="$itemExtensions" type="tagged" tag="api_platform.doctrine.orm.query_extension.item" />
    </service>
</services>
```

</code-selector>

Your data provider will now have access to the core extensions, here is an example on how to use them:

```php
<?php
// api/src/DataProvider/BlogPostItemDataProvider.php

namespace App\DataProvider;

use ApiPlatform\Core\Bridge\Doctrine\Orm\Extension\QueryResultItemExtensionInterface;
use ApiPlatform\Core\Bridge\Doctrine\Orm\Util\QueryNameGenerator;
use ApiPlatform\Core\DataProvider\ItemDataProviderInterface;
use ApiPlatform\Core\DataProvider\RestrictedDataProviderInterface;
use App\Entity\BlogPost;
use Doctrine\Persistence\ManagerRegistry;

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

## Community Data Providers

If you don't want to use the built-in Doctrine system, alternative approaches which offer an integration with API Platform exist.

* [Pomm Data Provider](https://github.com/pomm-project/pomm-api-platform): ([Pomm](http://www.pomm-project.org/) is a database access framework dedicated to PostgreSQL database.

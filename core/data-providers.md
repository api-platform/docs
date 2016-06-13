# Data providers

To retrieve data exposed by the API, API Platform uses classes called **data providers**. A data provider using [Doctrine
ORM](http://www.doctrine-project.org/projects/orm.html) to retrieve data from a database is included with the library and
is enabled by default. This data provider natively supports paged collections and filters. It can be used as is and fits
perfectly with common usages.

But sometime, you want to retrieve data from other sources such as a webservice, ElasticSearch, MongoDB or another ORM.
Custom data providers can be used to do so. A project can include as many data providers as it needs. The first able to retrieve data for a given resource will be used.

For a given resource, you can implement two kind of data providers:
- the [CollectionDataProvider](https://github.com/api-platform/core/blob/master/src/Api/CollectionDataProviderInterface.php) is used when fetching a collection.
- the [ItemDataProvider](https://github.com/api-platform/core/blob/master/src/Api/ItemDataProviderInterface.php) is used when fetching items.

In the following examples we will create custom data providers for the class `AppBundle\Model\BlogPost`.

## Custom collection data provider

First declare a Symfony service, for example:

```yaml
# AppBundle\Resources\config\services.yml

services:
    blog_post.collection_data_provider:
        class: 'AppBundle\DataProvider\BlogPostCollectionDataProvider'
        # data providers are using decorators (http://symfony.com/doc/master//components/dependency_injection/advanced.html#decorating-services)
        decorates: 'api_platform.collection_data_provider'
        arguments:
            - '@blog_post.collection_data_provider.inner'
```

Then, your `BlogPostCollectionDataProvider` has to implement the [`CollectionDataProviderInterface`](https://github.com/api-platform/core/blob/master/src/Api/CollectionDataProviderInterface.php):

The `getCollection` method must return an `array`, a `\Traversable` or a `PaginatorInterface` instance. If no data is available, you should return an empty array.

```php

// src/AppBundle/DataProvider/BlogPostCollectionDataProvider.php

namespace AppBundle\DataProvider;

use AppBundle\Entity\BlogPost;
use ApiPlatform\Core\Api\CollectionDataProviderInterface;
use ApiPlatform\Core\Exception\ResourceClassNotSupportedException;

class BlogPostCollectionDataProvider implements CollectionDataProviderInterface
{
    private $decorated;

    public function __construct(CollectionDataProviderInterface $decorated = null)
    {
        $this->decorated = $decorated;
    }

    /**
     * {@inheritdoc}
     */
    public function getCollection(string $resourceClass, string $operationName = null)
    {
        // tests if the class is supported by this data provider
        if (BlogPost::class !== $resourceClass) {
          throw new ResourceClassNotSupportedException();
        }

        return ['foo' => 'bar'];
    }
}
```

## Custom item data provider

Declare a Symfony service, for example:

```yaml

# AppBundle\Resources\config\services.yml

services:
    blog_post.item_data_provider:
        class: 'AppBundle\DataProvider\BlogPostItemDataProvider'
        decorates: 'api_platform.item_data_provider'
        arguments:
            - '@blog_post.item_data_provider.inner'
```

Then, your `BlogPostItemDataProvider` has to implement the [`ItemDataProviderInterface`](https://github.com/api-platform/core/blob/master/src/Api/ItemDataProviderInterface.php):

The `getItem` method can return `null` if no result has been found.

```php

// src/AppBundle/DataProvider/BlogPostItemDataProvider.php

namespace AppBundle\DataProvider;

use AppBundle\Entity\BlogPost;
use ApiPlatform\Core\Api\ItemDataProviderInterface;
use ApiPlatform\Core\Exception\ResourceClassNotSupportedException;

class BlogPostItemDataProvider implements ItemDataProviderInterface
{
    private $decorated;

    public function __construct(ItemDataProviderInterface $decorated = null)
    {
        $this->decorated = $decorated;
    }

    /**
     * {@inheritdoc}
     */
    public function getCollection(string $resourceClass, string $operationName = null)
    {
        if (BlogPost::class !== $resourceClass) {
          throw new ResourceClassNotSupportedException();
        }

        return (object) ['foo' => 'bar'];
    }
}
```

Previous chapter: [Operations](operations.md)<br>
Next chapter: [Filters](filters.md)

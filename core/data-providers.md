# Data Providers

To retrieve data exposed by the API, API Platform uses classes called **data providers**. A data provider using [Doctrine
ORM](http://www.doctrine-project.org/projects/orm.html) to retrieve data from a database is included with the library and
is enabled by default. This data provider natively supports paged collections and filters. It can be used as is and fits
perfectly with common usages.

But sometime, you want to retrieve data from other sources such as another persistence layer, a webservice, ElasticSearch
or MongoDB.
Custom data providers can be used to do so. A project can include as many data providers as it needs. The first able to
retrieve data for a given resource will be used.

For a given resource, you can implement two kind of interfaces:

* the [`CollectionDataProviderInterface`](https://github.com/api-platform/core/blob/master/src/DataProvider/CollectionDataProviderInterface.php)
  is used when fetching a collection.
* the [`ItemDataProviderInterface`](https://github.com/api-platform/core/blob/master/src/DataProvider/ItemDataProviderInterface.php)
  is used when fetching items.

In the following examples we will create custom data providers for an entity class class called `AppBundle\Entity\BlogPost`.
Note, that if your entity is not doctrine-related, you need to flag the identifier property by using `@ApiProperty(identifiier=true)` for things to work properly (see also [Entity Identifier Case](serialization-groups-and-relations.md#entity-identifier-case)).

## Custom Collection Data Provider

First, your `BlogPostCollectionDataProvider` has to implement the [`CollectionDataProviderInterface`](https://github.com/api-platform/core/blob/master/src/DataProvider/CollectionDataProviderInterface.php):

The `getCollection` method must return an `array`, a `Traversable` or a [`ApiPlatform\Core\DataProvider\PaginatorInterface`](https://github.com/api-platform/core/blob/master/src/DataProvider/PaginatorInterface.php) instance.
If no data is available, you should return an empty array.

```php
<?php

// src/AppBundle/DataProvider/BlogPostCollectionDataProvider.php

namespace AppBundle\DataProvider;

use AppBundle\Entity\BlogPost;
use ApiPlatform\Core\DataProvider\CollectionDataProviderInterface;
use ApiPlatform\Core\Exception\ResourceClassNotSupportedException;

final class BlogPostCollectionDataProvider implements CollectionDataProviderInterface
{
    public function getCollection(string $resourceClass, string $operationName = null)
    {
        if (BlogPost::class !== $resourceClass) {
            throw new ResourceClassNotSupportedException();
        }

        // Retrieve the blog post collection from somewhere
        return [new BlogPost(1), new BlogPost(2)];
    }
}
```

Then declare a Symfony service, for example:

```yaml
# app/config/services.yml

services:
    blog_post.collection_data_provider:
        class: 'AppBundle\DataProvider\BlogPostCollectionDataProvider'
        public: false
        tags:
            -  { name: 'api_platform.collection_data_provider', priority: 2 }
```

Tagging the service with the tag `api_platform.collection_data_provider` will enable API Platform Core to automatically
register and use this data provider. The optional attribute `priority` allows to define the order in wich are called the
data providers. The first data provider not throwing a `ApiPlatform\Core\Exception\ResourceClassNotSupportedException`
will be used.

## Custom Item Data Provider

The process is similar for item data providers. Create a `BlogPostItemDataProvider` implementing the [`ItemDataProviderInterface`](https://github.com/api-platform/core/blob/master/src/DataProvider/ItemDataProviderInterface.php)
interface:

The `getItem` method can return `null` if no result has been found.

```php
<?php

// src/AppBundle/DataProvider/BlogPostItemDataProvider.php

namespace AppBundle\DataProvider;

use AppBundle\Entity\BlogPost;
use ApiPlatform\Core\DataProvider\ItemDataProviderInterface;
use ApiPlatform\Core\Exception\ResourceClassNotSupportedException;

final class BlogPostItemDataProvider implements ItemDataProviderInterface
{
    public function getItem(string $resourceClass, $id, string $operationName = null, array $context = [])    {
        if (BlogPost::class !== $resourceClass) {
          throw new ResourceClassNotSupportedException();
        }

        // Retrieve the blog post item from somewhere
        return new BlogPost($id);
    }
}
```

The tag to use for item data providers is `api_platform.item_data_provider`. As for collection data providers, the `priority`
attribute can be used to order providers.

```yaml
# app/config/services.yml

services:
    blog_post.item_data_provider:
        class: 'AppBundle\DataProvider\BlogPostItemDataProvider'
        public: false
        tags:
            -  { name: 'api_platform.item_data_provider' }
```

Previous chapter: [Extending JSON-LD context](extending-jsonld-context.md)

Next chapter: [Extensions](extensions.md)

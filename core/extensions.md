# Extensions

API Platform Core provides a system to extend queries on items and collections.

Extensions are specific to Doctrine and Elasticsearch-PHP, and therefore, the Doctrine ORM / MongoDB ODM support or the Elasticsearch
reading support must be enabled to use this feature. If you use custom providers it's up to you to implement your own
extension system or not.

You can find a working example of a custom data provider using a pagination extension in the [API Platform's demo application](https://github.com/api-platform/demo/blob/master/api/src/DataProvider/Extension/TopBookPaginationExtension.php).

## Custom Doctrine ORM Extension

Custom extensions must implement the `ApiPlatform\Core\Bridge\Doctrine\Orm\Extension\QueryCollectionExtensionInterface` and / or the `ApiPlatform\Core\Bridge\Doctrine\Orm\Extension\QueryItemExtensionInterface` interfaces, to be run when querying for a collection of items and when querying for an item respectively.

If you use [custom data providers](data-providers.md), they must support extensions and be aware of active extensions to work properly.

### Example

In the following example, we will see how to always get the offers owned by the current user. We will set up an exception, whenever the user has the `ROLE_ADMIN`.

Given these two entities:

```php
<?php
// api/src/Entity/User.php

namespace App\Entity;

use ApiPlatform\Core\Annotation\ApiResource;

#[ApiResource]
class User
{
    // ...
}
```

```php
<?php
// api/src/Entity/Offer.php

namespace App\Entity;

use ApiPlatform\Core\Annotation\ApiResource;

#[ApiResource]
class Offer
{
    #[ORM\ManyToOne] 
    public User $user;

    //...
}
```

```php
<?php
// api/src/Doctrine/CurrentUserExtension.php

namespace App\Doctrine;

use ApiPlatform\Core\Bridge\Doctrine\Orm\Extension\QueryCollectionExtensionInterface;
use ApiPlatform\Core\Bridge\Doctrine\Orm\Extension\QueryItemExtensionInterface;
use ApiPlatform\Core\Bridge\Doctrine\Orm\Util\QueryNameGeneratorInterface;
use App\Entity\Offer;
use Doctrine\ORM\QueryBuilder;
use Symfony\Component\Security\Core\Security;

final class CurrentUserExtension implements QueryCollectionExtensionInterface, QueryItemExtensionInterface
{
    private $security;

    public function __construct(Security $security)
    {
        $this->security = $security;
    }

    public function applyToCollection(QueryBuilder $queryBuilder, QueryNameGeneratorInterface $queryNameGenerator, string $resourceClass, string $operationName = null): void
    {
        $this->addWhere($queryBuilder, $resourceClass);
    }

    public function applyToItem(QueryBuilder $queryBuilder, QueryNameGeneratorInterface $queryNameGenerator, string $resourceClass, array $identifiers, string $operationName = null, array $context = []): void
    {
        $this->addWhere($queryBuilder, $resourceClass);
    }

    private function addWhere(QueryBuilder $queryBuilder, string $resourceClass): void
    {
        if (Offer::class !== $resourceClass || $this->security->isGranted('ROLE_ADMIN') || null === $user = $this->security->getUser()) {
            return;
        }

        $rootAlias = $queryBuilder->getRootAliases()[0];
        $queryBuilder->andWhere(sprintf('%s.user = :current_user', $rootAlias));
        $queryBuilder->setParameter('current_user', $user->getId());
    }
}

```

Finally, if you're not using the autoconfiguration, you have to register the custom extension with either of those tags:

```yaml
# config/services.yaml
services:

    # ...

    'App\Doctrine\CurrentUserExtension':
        tags:
            - { name: api_platform.doctrine.orm.query_extension.collection }
            - { name: api_platform.doctrine.orm.query_extension.item }
```

The `api_platform.doctrine.orm.query_extension.collection` tag will register this service as a collection extension.
The `api_platform.doctrine.orm.query_extension.item` does the same thing for items.

Note that your extensions should have a positive priority if defined. Internal extensions have negative priorities, for reference:

| Service name                                               | Priority | Class                                              |
|------------------------------------------------------------|------|---------------------------------------------------------|
| `api_platform.doctrine.orm.query_extension.eager_loading` (collection) | -8 | ApiPlatform\Core\Bridge\Doctrine\Orm\Extension\EagerLoadingExtension |
| `api_platform.doctrine.orm.query_extension.eager_loading` (item) | -8 | ApiPlatform\Core\Bridge\Doctrine\Orm\Extension\EagerLoadingExtension |
| `api_platform.doctrine.orm.query_extension.filter` | -16 | ApiPlatform\Core\Bridge\Doctrine\Orm\Extension\FilterExtension |
| `api_platform.doctrine.orm.query_extension.filter_eager_loading` | -17 | ApiPlatform\Core\Bridge\Doctrine\Orm\Extension\FilterEagerLoadingExtension |
| `api_platform.doctrine.orm.query_extension.order` | -32 | ApiPlatform\Core\Bridge\Doctrine\Orm\Extension\OrderExtension |
| `api_platform.doctrine.orm.query_extension.pagination` | -64 | ApiPlatform\Core\Bridge\Doctrine\Orm\Extension\PaginationExtension |

#### Blocking Anonymous Users

This example adds a `WHERE` clause condition only when a fully authenticated user without `ROLE_ADMIN` tries to access a resource. It means that anonymous users will be able to access all data. To prevent this potential security issue, the API must ensure that the current user is authenticated.

To secure the access to endpoints, use the following access control rule:

```yaml
# config/package/security.yaml
security:
    # ...
    access_control:
        # ...
        - { path: ^/offers, roles: IS_AUTHENTICATED_FULLY }
        - { path: ^/users, roles: IS_AUTHENTICATED_FULLY }
```

## Custom Doctrine MongoDB ODM Extension

Creating custom extensions is the same as with Doctrine ORM.

The interfaces are:

* `ApiPlatform\Core\Bridge\Doctrine\MongoDbOdm\Extension\AggregationItemExtensionInterface` and `ApiPlatform\Core\Bridge\Doctrine\MongoDbOdm\Extension\AggregationCollectionExtensionInterface` to add stages to the [aggregation builder](https://www.doctrine-project.org/projects/doctrine-mongodb-odm/en/latest/reference/aggregation-builder.html).
* `ApiPlatform\Core\Bridge\Doctrine\MongoDbOdm\Extension\AggregationResultItemExtensionInterface` and `ApiPlatform\Core\Bridge\Doctrine\MongoDbOdm\Extension\AggregationResultCollectionExtensionInterface` to return a result.

The tags are `api_platform.doctrine_mongodb.odm.aggregation_extension.item` and `api_platform.doctrine_mongodb.odm.aggregation_extension.collection`.

The custom extensions receive the [aggregation builder](https://www.doctrine-project.org/projects/doctrine-mongodb-odm/en/latest/reference/aggregation-builder.html),
used to execute [complex operations on data](https://docs.mongodb.com/manual/aggregation/).

## Custom Elasticsearch Extension

Currently only extensions querying for a collection of items through a [search request](https://www.elastic.co/guide/en/elasticsearch/reference/current/search-request-body.html)
are supported. So your custom extensions must implement the `RequestBodySearchCollectionExtensionInterface`. Register your
custom extensions as services and tag them with the `api_platform.elasticsearch.request_body_search_extension.collection` tag.

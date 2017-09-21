# Extensions

API Platform Core provides a system to extend queries on items and collections.

Extensions are specific to Doctrine, and therefore, the Doctrine ORM support must be enabled to use this feature. If you use custom providers it's up to you to implement your own extension system or not.

## Custom Extension

Custom extensions must implement the `ApiPlatform\Core\Bridge\Doctrine\Orm\Extension\QueryCollectionExtensionInterface` and / or the `ApiPlatform\Core\Bridge\Doctrine\Orm\Extension\QueryItemExtensionInterface` interfaces, to be run when querying for a collection of items and when querying for an item respectively.

If you use [custom data providers](data-providers.md), they must support extensions and be aware of active extensions to work properly.

## Example

In the following example, we will see how to always get the offers owned by the current user. We will set up an exception, whenever the user has the `ROLE_ADMIN`.

Given these two entities:

```php
<?php
// src/AppBundle/Entity/User.php

namespace AppBundle\Entity;

use ApiPlatform\Core\Annotation\ApiResource;

/**
 * @ApiResource
 */
class User
{
    // ...
}
```

```php
<?php
// src/AppBundle/Entity/Offer.php

namespace AppBundle\Entity;

use ApiPlatform\Core\Annotation\ApiResource;

/**
 * @ApiResource
 */
class Offer
{
   /**
     * @var User
     * @ORM\ManyToOne(targetEntity="User")
     */
    private $user;

    //...
}
```

```php
<?php
// src/AppBundle/Doctrine/ORM/Extension/CurrentUserExtension.php

namespace AppBundle\Doctrine\ORM\Extension;

use ApiPlatform\Core\Bridge\Doctrine\Orm\Extension\QueryCollectionExtensionInterface;
use ApiPlatform\Core\Bridge\Doctrine\Orm\Extension\QueryItemExtensionInterface;
use ApiPlatform\Core\Bridge\Doctrine\Orm\Util\QueryNameGeneratorInterface;
use AppBundle\Entity\Offer;
use AppBundle\Entity\User;
use Doctrine\ORM\QueryBuilder;
use Symfony\Component\Security\Core\Authentication\Token\Storage\TokenStorageInterface;
use Symfony\Component\Security\Core\Authorization\AuthorizationChecker;

final class CurrentUserExtension implements QueryCollectionExtensionInterface, QueryItemExtensionInterface
{
    private $tokenStorage;
    private $authorizationChecker;

    public function __construct(TokenStorageInterface $tokenStorage, AuthorizationChecker $checker)
    {
        $this->tokenStorage = $tokenStorage;
        $this->authorizationChecker = $checker;
    }

    /**
     * {@inheritdoc}
     */
    public function applyToCollection(QueryBuilder $queryBuilder, QueryNameGeneratorInterface $queryNameGenerator, string $resourceClass, string $operationName = null)
    {
        $this->addWhere($queryBuilder, $resourceClass);
    }

    /**
     * {@inheritdoc}
     */
    public function applyToItem(QueryBuilder $queryBuilder, QueryNameGeneratorInterface $queryNameGenerator, string $resourceClass, array $identifiers, string $operationName = null, array $context = [])
    {
        $this->addWhere($queryBuilder, $resourceClass);
    }

    /**
     *
     * @param QueryBuilder $queryBuilder
     * @param string       $resourceClass
     */
    private function addWhere(QueryBuilder $queryBuilder, string $resourceClass)
    {
        $user = $this->tokenStorage->getToken()->getUser();
        if ($user instanceof User && Offer::class === $resourceClass && !$this->authorizationChecker->isGranted('ROLE_ADMIN')) {
            $rootAlias = $queryBuilder->getRootAliases()[0];
            $queryBuilder->andWhere(sprintf('%s.user = :current_user', $rootAlias));
            $queryBuilder->setParameter('current_user', $user->getId());
        }
    }
}

```

Finally register the custom extension:

```yaml
# app/config/services.yml

services:

    # ...

    'AppBundle\Doctrine\ORM\Extension\CurrentUserExtension':
        tags:
            - { name: api_platform.doctrine.orm.query_extension.collection, priority: 9 }
            - { name: api_platform.doctrine.orm.query_extension.item }
```

Thanks to the `api_platform.doctrine.orm.query_extension.collection` tag, API Platform will register this service as a collection extension. The `api_platform.doctrine.orm.query_extension.item` do the same thing for items.

Notice the priority level for the `api_platform.doctrine.orm.query_extension.collection` tag. When an extension implements the `ApiPlatform\Core\Bridge\Doctrine\Orm\Extension\QueryResultCollectionExtensionInterface` or the `ApiPlatform\Core\Bridge\Doctrine\Orm\Extension\QueryResultItemExtensionInterface` interface to return results by itself, any lower priority extension will not be executed. Because the pagination is enabled by default with a priority of 8, the priority of the `app.doctrine.orm.query_extension.current_user` service must be at least 9 to ensure its execution.

### Blocking Anonymous Users

This example adds a `WHERE` clause condition only when a fully authenticated user without `ROLE_ADMIN` tries to access to a resource. It means that anonymous users will be able to access to all data. To prevent this potential security issue, the API must ensure that the current user is authenticated.

To secure the access to endpoints, use the following access control rule:

```yaml
# app/config/security.yml

security:

    # ...

    access_control:

        # ...

        - { path: ^/offers, roles: IS_AUTHENTICATED_FULLY }
        - { path: ^/users, roles: IS_AUTHENTICATED_FULLY }
```

Previous chapter: [Data Providers](data-providers.md)

Next chapter: [Security](security.md)

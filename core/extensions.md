# Extensions

API Platform Core provides a system to extend queries on items and collections.

Extensions are specific to Doctrine, and therefor, the Doctrine ORM support must be enabled.
If you use custom providers, they should support extensions and be aware of active extensions OR implement their own extension systems.

## Custom Extension

Adding an extension is as easy as registering a service in your `app/config/services.yml` file and create the class you need.

Custom extension must implement the `ApiPlatform\Core\Bridge\Doctrine\Orm\Extension\QueryCollectionExtensionInterface`
and / or the `ApiPlatform\Core\Bridge\Doctrine\Orm\Extension\QueryItemExtensionInterface`
interfaces, depending if you are asking for a collection of items or just an item.

If you use [custom data providers](data-providers.md), they must support extensions and be aware of active extensions to work
properly.

## Filter upon the current user

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
use ApiPlatform\Core\Metadata\Property\Factory\PropertyMetadataFactoryInterface;
use ApiPlatform\Core\Metadata\Property\Factory\PropertyNameCollectionFactoryInterface;
use AppBundle\Entity\Offer;
use AppBundle\Entity\User;
use Doctrine\ORM\QueryBuilder;
use Doctrine\ORM\Query\Expr;
use Symfony\Component\Security\Core\Authentication\Token\Storage\TokenStorageInterface;
use Symfony\Component\Security\Core\Authorization\AuthorizationChecker;

final class CurrentUserExtension implements QueryCollectionExtensionInterface, QueryItemExtensionInterface
{
    private $authorizationChecker;
    private $propertyNameCollectionFactory;
    private $propertyMetadataFactory;
    private $token;

    public function __construct(PropertyNameCollectionFactoryInterface $propertyNameCollectionFactory, PropertyMetadataFactoryInterface $propertyMetadataFactory, TokenStorageInterface $token, AuthorizationChecker $checker)
    {
        $this->propertyMetadataFactory = $propertyMetadataFactory;
        $this->propertyNameCollectionFactory = $propertyNameCollectionFactory;
        $this->token = $token;
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
    public function applyToItem(QueryBuilder $queryBuilder, QueryNameGeneratorInterface $queryNameGenerator, string $resourceClass, array $identifiers, string $operationName = null)
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
        $user = $this->token->getToken()->getUser();
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
    app.doctrine.orm.query_extension.current_user:
        class: AppBundle\Doctrine\ORM\Extension\CurrentUserExtension
        public: false
        arguments:
            - '@api_platform.metadata.property.name_collection_factory'
            - '@api_platform.metadata.property.metadata_factory'
            - '@security.token_storage'
            - '@security.authorization_checker'
        tags:
            - { name: api_platform.doctrine.orm.query_extension.collection, priority: 9 }
            - { name: api_platform.doctrine.orm.query_extension.item }
```

Thanks to the api_platform.doctrine.orm.query_extension.collection tag, API Platform will register this service as a collection extension.
The api_platform.doctrine.orm.query_extension.item do the same thing for items.

Notice the priority level for the Collection tag.
There is a case, when an extension implements the `ApiPlatform\Core\Bridge\Doctrine\Orm\Extension\QueryResultCollectionExtensionInterface` or the `ApiPlatform\Core\Bridge\Doctrine\Orm\Extension\QueryResultItemExtensionInterface` and supports to immediately return results,
any lower priority extension will not be executed.
In our case, since the pagination is activated by default ([see how to disable the pagination](pagination.md#disabling-the-pagination)) and the `ApiPlatform\Core\Bridge\Doctrine\Orm\Extension\PaginationExtension` is declared with a priority 8, we must declare a priority to at least 9 to ensure it's execution.

Previous chapter: [Filters](filters.md)

Next chapter: [Serialization Groups and Relations](serialization-groups-and-relations.md)
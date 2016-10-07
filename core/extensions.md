# Extensions

API Platform Core provides a system to extends queries on items and collections readings.
You can create custom extension that fit your needs.

## Custom Extension

If Doctrine ORM support is enabled, adding extension is as easy as registering a service in your `app/config/services.yml` file and create the extension you need.

Custom extension can be written by implementing the `ApiPlatform\Core\Bridge\Doctrine\Orm\Extension\QueryCollectionExtensionInterface`
and / or the `ApiPlatform\Core\Bridge\Doctrine\Orm\Extension\QueryItemExtensionInterface`
interfaces, depending on your whether you asking for a collection of items or just an item.

If you use [custom data providers](data-providers.md), they must support extensions and be aware of active extensions to work
properly.

## Filter upon the current user

In the following example, we will see how to always get the offers owned by the current user. We will set up an exception, whenever the user possesses the `ROLE_ADMIN`.
Given these two entities :

```php
<?php
// src/AppBundle/Entity/User.php

namespace AppBundle\Entity;

use ApiPlatform\Core\Annotation\ApiResource;

/**
 * @ApiResource()
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
 * @ApiResource()
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
// src/AppBundle/Doctrine/ORM/Extension/CurrentUserExtension.php
<?php

namespace AppBundle\Doctrine\ORM\Extension;

use ApiPlatform\Core\Bridge\Doctrine\Orm\Extension\QueryCollectionExtensionInterface;
use ApiPlatform\Core\Bridge\Doctrine\Orm\Extension\QueryItemExtensionInterface;
use ApiPlatform\Core\Bridge\Doctrine\Orm\Util\QueryNameGeneratorInterface;
use ApiPlatform\Core\Metadata\Property\Factory\PropertyMetadataFactoryInterface;
use ApiPlatform\Core\Metadata\Property\Factory\PropertyNameCollectionFactoryInterface;
use AppBundle\Entity\Offer;
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
     * Add where condition
     *
     * @param QueryBuilder $queryBuilder
     * @param string       $resourceClass
     */
    private function addWhere(QueryBuilder $queryBuilder, string $resourceClass)
    {
        if (Offer::class === $resourceClass && !$this->authorizationChecker->isGranted('ROLE_ADMIN')) {
            $user = $this->token->getToken()->getUser();
            $rootAlias = $queryBuilder->getRootAliases()[0];
            $queryBuilder->andWhere(sprintf('%s.user = ', $rootAlias).$user->getId());
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
            - { name: api_platform.doctrine.orm.query_extension.collection, priority: 64 }
            - { name: api_platform.doctrine.orm.query_extension.item, priority: 64 }
```

Having the item related tag and interface, allows you to customize the query when trying to get/read a specific Item.
Having the collection related tag and interface, allows you to customize the query when trying to get/read a collection of items.

Previous chapter: [Filters](filters.md)

Next chapter: [Serialization Groups and Relations](serialization-groups-and-relations.md)
# Create a user-aware filter

_Note: This guide assumes you are using Doctrine ORM as your persistence system._

Doctrine ORM features [a filter system](http://docs.doctrine-project.org/projects/doctrine-orm/en/latest/reference/filters.html) that allows the developer to add SQL to the conditional clauses of queries, regardless of the place where the SQL is generated (e.g. from a DQL query, or by loading associated entities).
These are applied on collections and items and therefore are incredibly useful.

The following information, specific to Doctrine filters in Symfony, is based upon [a great article posted on Michaël Perrin's blog](http://blog.michaelperrin.fr/2014/12/05/doctrine-filters/).

Suppose we have a `User` entity and an `Order` entity related to the `User` one. A user should only see his orders and no one else's.

```php
<?php
// api/src/Entity/User.php

namespace App\Entity;

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
// api/src/Entity/Order.php

namespace App\Entity;

use ApiPlatform\Core\Annotation\ApiResource;
use Doctrine\ORM\Mapping as ORM;

/**
 * @ApiResource
 */
class Order
{
    // ...

    /**
     * @ORM\ManyToOne(targetEntity="User")
     * @ORM\JoinColumn(name="user_id", referencedColumnName="id")
     **/
    public $user;
    
    // ...
}
```

The whole idea is that any query on the order table should add a `WHERE user_id = :user_id` condition.

Start by creating a custom annotation to mark restricted entities:

```php
<?php
// api/Annotation/UserAware.php

namespace App\Annotation;

use Doctrine\Common\Annotations\Annotation;

/**
 * @Annotation
 * @Target("CLASS")
 */
final class UserAware
{
    public $userFieldName;
}
```

Then, let's mark the `Order` entity as a "user aware" entity.

```php
<?php
// api/src/Entity/Order.php

namespace App\Entity;

use App\Annotation\UserAware;

/**
 * @UserAware(userFieldName="user_id")
 */
class Order {
    // ...
}
```

Now, create a Doctrine filter class:

```php
<?php
// api/src/Filter/UserFilter.php

namespace App\Filter;

use App\Annotation\UserAware;
use Doctrine\ORM\Mapping\ClassMetaData;
use Doctrine\ORM\Query\Filter\SQLFilter;
use Doctrine\Common\Annotations\Reader;

final class UserFilter extends SQLFilter
{
    private $reader;

    public function addFilterConstraint(ClassMetadata $targetEntity, string $targetTableAlias): string
    {
        if (null === $this->reader) {
            throw new \RuntimeException(sprintf('An annotation reader must be provided. Be sure to call "%s::setAnnotationReader()".', __CLASS__));
        }

        // The Doctrine filter is called for any query on any entity
        // Check if the current entity is "user aware" (marked with an annotation)
        $userAware = $this->reader->getClassAnnotation($targetEntity->getReflectionClass(), UserAware::class);
        if (!$userAware) {
            return '';
        }

        $fieldName = $userAware->userFieldName;
        try {
            // Don't worry, getParameter automatically escapes parameters
            $userId = $this->getParameter('id');
        } catch (\InvalidArgumentException $e) {
            // No user id has been defined
            return '';
        }

        if (empty($fieldName) || empty($userId)) {
            return '';
        }

        return sprintf('%s.%s = %s', $targetTableAlias, $fieldName, $userId);
    }

    public function setAnnotationReader(Reader $reader): void
    {
        $this->reader = $reader;
    }
}
```

Now, we must configure the Doctrine filter.

```yaml
# api/config/packages/api_platform.yaml
doctrine:
    orm:
        filters:
            user_filter:
                class: App\Filter\UserFilter
```

Add a listener for every request that initializes the Doctrine filter with the current user in your bundle services declaration file.

```yaml
# api/config/services.yaml
services:
    # ...
    'App\EventListener\UserFilterConfigurator':
        tags:
            - { name: kernel.event_listener, event: kernel.request, priority: 5 }
        # Autoconfiguration must be disabled to set a custom priority
        autoconfigure: false
```

It's key to set the priority higher than the `ApiPlatform\Core\EventListener\ReadListener`'s priority, as flagged in [this issue](https://github.com/api-platform/core/issues/1185), as otherwise the `PaginatorExtension` will ignore the Doctrine filter and return incorrect `totalItems` and `page` (first/last/next) data.

Lastly, implement the configurator class:

```php
<?php
// api/EventListener/UserFilterConfigurator.php

namespace App\EventListener;

use Symfony\Component\Security\Core\User\UserInterface;
use Symfony\Component\Security\Core\Authentication\Token\Storage\TokenStorageInterface;
use Doctrine\ORM\EntityManagerInterface;
use Doctrine\Common\Annotations\Reader;

final class UserFilterConfigurator
{
    private $em;
    private $tokenStorage;
    private $reader;

    public function __construct(EntityManagerInterface $em, TokenStorageInterface $tokenStorage, Reader $reader)
    {
        $this->em = $em;
        $this->tokenStorage = $tokenStorage;
        $this->reader = $reader;
    }

    public function onKernelRequest(): void
    {
        if (!$user = $this->getUser()) {
            throw new \RuntimeException('There is no authenticated user.');
        }

        $filter = $this->em->getFilters()->enable('user_filter');
        $filter->setParameter('id', $user->getId());
        $filter->setAnnotationReader($this->reader);
    }

    private function getUser(): ?UserInterface
    {
        if (!$token = $this->tokenStorage->getToken()) {
            return null;
        }

        $user = $token->getUser();
        return $user instanceof UserInterface ? $user : null;
    }
}
```

Done: Doctrine will automatically filter all "UserAware" entities!

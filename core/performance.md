# Performance

## Enabling the Metadata Cache

Computing metadata used by the bundle is a costly operation. Fortunately, metadata can be computed once and then cached.
API Platform internally uses a [PSR-6](http://www.php-fig.org/psr/psr-6/) cache. If the Symfony Cache Component is available
(the default in the official distribution), it automatically enables the support for the best cache adapter available.

Best performance is achieved using [APCu](https://github.com/krakjoe/apcu). Be sure to have the APCu extension installed
on your production server, API Platform will automatically use it.

## Using PPM (PHP-PM)

Response time of the API can be improved up to 15x by using [PHP Process Manager](https://github.com/php-pm/php-pm). If
you want to use it on your project, follow the documentation dedicated to Symfony on the PPM website.

Keep in mind that PPM is still in an early stage of development and can cause issues in production.

## Doctrine Queries and Indexes

### Search Filter

When using the `SearchFilter` and case insensivity, Doctrine will use the `LOWER` SQL function. Depending on your
driver, you may want to carefully index it by using a [function-based
index](http://use-the-index-luke.com/sql/where-clause/functions/case-insensitive-search) or it will impact performanc
with a huge collection. [Here are some examples to index LIKE
filters](http://use-the-index-luke.com/sql/where-clause/searching-for-ranges/like-performance-tuning) depending on your
database driver.

### Eager loading

By default Doctrine comes with [lazy loading](http://doctrine-orm.readthedocs.io/en/latest/reference/working-with-objects.html#by-lazy-loading).
Usually a killer time-saving feature and also a performance killer with large applications.

Fortunately, Doctrine proposes another approach to remedy this problem: [eager loading](http://doctrine-orm.readthedocs.io/en/latest/reference/working-with-objects.html#by-eager-loading).
This can easily be enabled for a relation: `@ORM\ManyToOne(fetch="EAGER")`.

By default in API Platform, we made the choice to force eager loading for all relations, with or without the Doctrine
`fetch` attribute. Thanks to the eager loading [extension](extensions.md).

#### Max joins

There is a default restriction with this feature. We allow up to 30 joins per query. Beyond, an 
`ApiPlatform\Core\Exception\RuntimeException` exception will be thrown but this value can easily be increased with a
little of configuration:

```yaml
# app/config/config.yaml

api_platform:
    # ...

    eager_loading:
        max_joins: 100

    # ...
```

Be careful when you exceed this limit, it's often caused by the result of a circular reference. [Serializer groups](serialization-groups-and-relations.md)
can be a good solution to fix this issue.

#### Force eager

As mentioned above, by default we force eager loading for all relations. This behaviour can be modified with the 
configuration in order to apply it only on join relations having the `EAGER` fetch mode:

```yaml
# app/config/config.yaml

api_platform:
    # ...

    eager_loading:
        force_eager: false

    # ...
```

#### Override at resource and operation level

When eager loading is enabled, whatever the status of the `force_eager` parameter, you can easily override it directly
from the configuration of each resource. You can do this at the resource level, at the operations level, or both:

```php
<?php
// src/AppBundle/Entity/Address.php

namespace AppBundle\Entity;

use ApiPlatform\Core\Annotation\ApiResource;
use Doctrine\ORM\Mapping as ORM;

/**
 * @ApiResource
 * @ORM\Entity
 */
class Address
{
    // ...
}
```

```php
<?php
// src/AppBundle/Entity/User.php

namespace AppBundle\Entity;

use ApiPlatform\Core\Annotation\ApiResource;
use Doctrine\ORM\Mapping as ORM;

/**
 * @ApiResource(attributes={"force_eager"=false})
 * @ORM\Entity
 */
class User
{
    /**
     * @var Address
     * 
     * @ORM\ManyToOne(targetEntity="Address", fetch="EAGER") 
     */
    public $address;

    /**
     * @var Group[]
     *
     * @ORM\ManyToMany(targetEntity="Group", inversedBy="users")
     * @ORM\JoinTable(name="users_groups")
     */
    public $groups;
}
```

```php
<?php
// src/AppBundle/Entity/Group.php

namespace AppBundle\Entity;

use ApiPlatform\Core\Annotation\ApiResource;
use Doctrine\ORM\Mapping as ORM;

/**
 * @ApiResource(
 *     attributes={"force_eager"=false},
 *     itemOperations={
 *         "get"={"method"="GET", "force_eager"=true},
 *         "post"={"method"="POST"}
 *     },
 *     collectionOperations={
 *         "get"={"method"="GET", "force_eager"=true},
 *         "post"={"method"="POST"}
 *     }
 * )
 * @ORM\Entity
 */
class Group
{
    /**
     * @var User[]
     *
     * @ManyToMany(targetEntity="User", mappedBy="groups")
     */
    public $users;
}
```

Be careful, the operation level is higher priority than the resource level but both are higher priority than the global
configuration.

#### Disable eager loading

If for any reason you don't want the eager loading feature, you can turn off it in the configuration:

```yaml
# app/config/config.yaml

api_platform:
    # ...

    eager_loading:
        enabled: false

    # ...
```

The whole configuration seen before will no longer work and Doctrine will recover its default behavior.

Previous chapter: [Security](security.md)

Next chapter: [Operation Path Naming](operation-path-naming.md)

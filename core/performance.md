# Performance

## Enabling the Built-in HTTP Cache Invalidation System

Exposing a hypermedia API has [many advantages](http://blog.theamazingrando.com/in-band-vs-out-of-band.html). One of
them is the ability to know exactly which resources are included in HTTP responses created by the API. We used this
specificity to make API Platform apps blazing fast.

When the cache mechanism [is enabled](configuration.md), API Platform collects identifiers of every resources
included in a given HTTP response (including lists, embedded documents and subresources) and returns them in a special
HTTP header called [Cache-Tags](https://support.cloudflare.com/hc/en-us/articles/206596608-How-to-Purge-Cache-Using-Cache-Tags-Enterprise-only-).

A [cache reverse proxy](https://en.wikipedia.org/wiki/Web_accelerator) supporting cache tags (Varnish, CloudFlare,
Fastly…) must be put in front of the web server and store all responses returned by the API with a high
[TTL](https://en.wikipedia.org/wiki/Time_to_live). When a resource is modified, API Platform takes care of purging all
responses containing it in the proxy’s cache. It means that after the first request, all subsequent requests will not
touch the web server, and will be served instantly from the cache. It also means that the content served will always be
fresh, because the cache is purged in real time.

The support for most specific cases such as the invalidation of collections when a document is added or removed or for
relationships and inverse relations is built-in.

We also included [Varnish](https://varnish-cache.org/) in the [Docker setup](../deployment/docker.md) provided with the
distribution of API Platform, so this feature works out of the box.

Integration with Varnish and the Doctrine ORM is shipped with the core library. You can easily implement the support for
any other proxy or persistence system.

### Extending Cache-Tags for invalidation

Sometimes you need individual resources like `/me`. To work properly with Varnish, the cache tags need to be augmented with these resources. Here is an example how this can be done:

```php
<?php

declare(strict_types=1);

namespace App\EventSubscriber;

use ApiPlatform\Core\EventListener\EventPriorities;
use App\Entity\User;
use Symfony\Component\EventDispatcher\EventSubscriberInterface;
use Symfony\Component\HttpKernel\Event\GetResponseEvent;
use Symfony\Component\HttpKernel\KernelEvents;

final class UserResourcesSubscriber implements EventSubscriberInterface
{
    public static function getSubscribedEvents()
    {
        return [
            KernelEvents::REQUEST => ['extendResources', EventPriorities::POST_READ]
        ];
    }

    public function extendResources(GetResponseEvent $event)
    {
        $request = $event->getRequest();
        $class = $request->attributes->get('_api_resource_class');

        if ($class === User::class) {
            $resources = [
                '/me'
            ];

            $request->attributes->set('_resources', $request->attributes->get('_resources', []) + (array)$resources);
        }
    }
}
```

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
index](http://use-the-index-luke.com/sql/where-clause/functions/case-insensitive-search) or it will impact performance
with a huge collection. [Here are some examples to index LIKE
filters](http://use-the-index-luke.com/sql/where-clause/searching-for-ranges/like-performance-tuning) depending on your
database driver.

### Eager Loading

By default Doctrine comes with [lazy loading](http://doctrine-orm.readthedocs.io/en/latest/reference/working-with-objects.html#by-lazy-loading).
Usually a killer time-saving feature and also a performance killer with large applications.

Fortunately, Doctrine proposes another approach to remedy this problem: [eager loading](http://doctrine-orm.readthedocs.io/en/latest/reference/working-with-objects.html#by-eager-loading).
This can easily be enabled for a relation: `@ORM\ManyToOne(fetch="EAGER")`.

By default in API Platform, we made the choice to force eager loading for all relations, with or without the Doctrine
`fetch` attribute. Thanks to the eager loading [extension](extensions.md). The `EagerLoadingExtension` will join every
readable association according to the serialization context. If you want to fetch an association that is not serializable
you've to bypass `readable` and `readableLink` by using the `fetchEager` attribute on the property declaration, for example:

```php
/**
 * @ApiProperty(attributes={"fetchEager": true})
 */
 public $foo;
```

#### Max Joins

There is a default restriction with this feature. We allow up to 30 joins per query. Beyond, an
`ApiPlatform\Core\Exception\RuntimeException` exception will be thrown but this value can easily be increased with a
little of configuration:

```yaml
# api/config/packages/api_platform.yaml
api_platform:
    eager_loading:
        max_joins: 100
```

Be careful when you exceed this limit, it's often caused by the result of a circular reference. [Serializer groups](serialization.md)
can be a good solution to fix this issue.

#### Force Eager

As mentioned above, by default we force eager loading for all relations. This behaviour can be modified with the
configuration in order to apply it only on join relations having the `EAGER` fetch mode:

```yaml
# api/config/packages/api_platform.yaml
api_platform:
    eager_loading:
        force_eager: false
```

#### Override at Resource and Operation Level

When eager loading is enabled, whatever the status of the `force_eager` parameter, you can easily override it directly
from the configuration of each resource. You can do this at the resource level, at the operations level, or both:

```php
<?php
// api/src/Entity/Address.php

namespace App\Entity;

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
// api/src/Entity/User.php

namespace App\Entity;

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
// api/src/Entity/Group.php

namespace App\Entity;

use ApiPlatform\Core\Annotation\ApiResource;
use Doctrine\ORM\Mapping as ORM;

/**
 * @ApiResource(
 *     attributes={"force_eager"=false},
 *     itemOperations={
 *         "get"={"force_eager"=true},
 *         "post"
 *     },
 *     collectionOperations={
 *         "get"={"force_eager"=true},
 *         "post"
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

#### Disable Eager Loading

If for any reason you don't want the eager loading feature, you can turn it off in the configuration:

```yaml
# api/config/packages/api_platform.yaml
api_platform:
    eager_loading:
        enabled: false
```

The whole configuration seen before will no longer work and Doctrine will recover its default behavior.

### Partial Pagination

When using the default pagination, the Doctrine paginator will execute a `COUNT` query on the collection. The result of the
`COUNT` query is used to compute the latest page available. With big collections this can lead to quite long response times.
If you don't mind not having the latest page available, you can enable partial pagination and avoid the `COUNT` query:

```yaml
# api/config/packages/api_platform.yaml
api_platform:
    collection:
        pagination:
            partial: true # Disabled by default
```

More details are available on the [pagination documentation](pagination.md#partial-pagination).

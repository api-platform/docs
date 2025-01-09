# Performance and Cache

## Enabling the Built-in HTTP Cache Invalidation System

Exposing a hypermedia API has [many advantages](http://blog.theamazingrando.com/in-band-vs-out-of-band.html). One of them
is the ability to know exactly which resources are included in HTTP responses created by the API. We used this specificity
to make API Platform apps blazing fast.

When the cache mechanism [is enabled](configuration.md), API Platform collects identifiers of every resource included in
a given HTTP response (including lists, embedded documents and subresources) and returns them in a special HTTP header
called [Cache-Tags](https://support.cloudflare.com/hc/en-us/articles/206596608-How-to-Purge-Cache-Using-Cache-Tags-Enterprise-only-).

A caching [reverse proxy](https://en.wikipedia.org/wiki/Reverse_proxy) supporting cache tags (e.g. Varnish, Cloudflare,
Fastly) must be put in front of the web server and store all responses returned by the API with a high [TTL](https://en.wikipedia.org/wiki/Time_to_live).
This means that after the first request, all subsequent requests will not hit the web server, and will be served instantly
from the cache.

When a resource is modified, API Platform takes care of purging all responses containing it in the proxy’s
cache. This ensures that the content served will always be fresh, because the cache is purged in real time. Support for
most specific cases such as the invalidation of collections when a document is added or removed or for relationships and
inverse relations is built-in.

Integration with Varnish and Doctrine ORM is shipped with the core library, and [Varnish](https://varnish-cache.org/) is
included in the Docker setup provided with the [API Platform distribution](../distribution/index.md). If you use the distribution,
this feature works out of the box.

If you don't use the distribution, add the following configuration to enable the cache invalidation system:

```yaml
api_platform:
    http_cache:
        invalidation:
            enabled: true
            varnish_urls: ['%env(VARNISH_URL)%']
        public: true
    defaults:
        cache_headers:
            max_age: 0
            shared_max_age: 3600
            vary: ['Content-Type', 'Authorization', 'Origin']
```

Support for reverse proxies other than Varnish can easily be added by implementing the `ApiPlatform\Core\HttpCache\PurgerInterface`.

In addition to the cache invalidation mechanism, you may want to [use HTTP/2 Server Push to pre-emptively send relations
to the client](push-relations.md).

### Extending Cache-Tags for Invalidation

Sometimes you need individual resources like `/me`. To work properly with Varnish, the `Cache-Tags` header needs to be
augmented with these resources. Here is an example of how this can be done:

```php
<?php

declare(strict_types=1);

namespace App\EventSubscriber;

use ApiPlatform\Core\EventListener\EventPriorities;
use App\Entity\User;
use Symfony\Component\EventDispatcher\EventSubscriberInterface;
use Symfony\Component\HttpKernel\Event\RequestEvent;
use Symfony\Component\HttpKernel\KernelEvents;

final class UserResourcesSubscriber implements EventSubscriberInterface
{
    public static function getSubscribedEvents()
    {
        return [
            KernelEvents::REQUEST => ['extendResources', EventPriorities::POST_READ]
        ];
    }

    public function extendResources(RequestEvent $event): void
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

## Setting Custom HTTP Cache Headers

The `cache_headers` attribute can be used to set custom HTTP cache headers:

```php
use ApiPlatform\Core\Annotation\ApiResource;

#[ApiResource(cacheHeaders: [
    "max_age" => 60,
    "shared_max_age" => 120,
    "vary" => ["Authorization", "Accept-Language"]
])]
class Book
{
    // ...
}
```

For all endpoints related to this resource class, the following HTTP headers will be set:

```http
Cache-Control: max-age=60, public, s-maxage=120
Vary: Authorization, Accept-Language
```

It's also possible to set different cache headers per operation:

```php
use ApiPlatform\Core\Annotation\ApiResource;

#[ApiResource(
    itemOperations: [
        "get" => [
            "cache_headers" => ["max_age" => 60, "shared_max_age" => 120],
        ]
    ]
  )
class Book
{
    // ...
}
```

## Enabling the Metadata Cache

Computing metadata used by the bundle is a costly operation. Fortunately, metadata can be computed once and then cached.
API Platform internally uses a [PSR-6](http://www.php-fig.org/psr/psr-6/) cache. If the Symfony Cache component is available
(the default in the API Platform distribution), it automatically enables support for the best cache adapter available.

Best performance is achieved using [APCu](https://github.com/krakjoe/apcu). Be sure to have the APCu extension installed
on your production server. API Platform will automatically use it.

## Using PPM (PHP-PM)

Response time of the API can be improved up to 15x by using [PHP Process Manager](https://github.com/php-pm/php-pm). If
you want to use it on your project, follow the documentation dedicated to Symfony on the PPM website.

Keep in mind that PPM is still in an early stage of development and can cause issues in production.

## Doctrine Queries and Indexes

### Search Filter

When using the `SearchFilter` and case insensitivity, Doctrine will use the `LOWER` SQL function. Depending on your
driver, you may want to carefully index it by using a [function-based
index](http://use-the-index-luke.com/sql/where-clause/functions/case-insensitive-search) or it will impact performance
with a huge collection. [Here are some examples to index LIKE
filters](http://use-the-index-luke.com/sql/where-clause/searching-for-ranges/like-performance-tuning) depending on your
database driver.

### Eager Loading

By default Doctrine comes with [lazy loading](https://www.doctrine-project.org/projects/doctrine-orm/en/latest/reference/working-with-objects.html#by-lazy-loading) - usually a killer time-saving feature but also a performance killer with large applications.

Fortunately, Doctrine offers another approach to solve this problem: [eager loading](https://www.doctrine-project.org/projects/doctrine-orm/en/current/reference/working-with-objects.html#by-eager-loading).
This can easily be enabled for a relation: `#[ORM\ManyToOne(fetch: "EAGER")]`.

By default in API Platform, we made the choice to force eager loading for all relations, with or without the Doctrine
`fetch` attribute. Thanks to the eager loading [extension](extensions.md). The `EagerLoadingExtension` will join every
readable association according to the serialization context. If you want to fetch an association that is not serializable,
you have to bypass `readable` and `readableLink` by using the `fetchEager` attribute on the property declaration, for example:

```php
#[ApiProperty(fetchEager: true)]
 public $foo;
```

#### Max Joins

There is a default restriction with this feature. We allow up to 30 joins per query. Beyond that, an
`ApiPlatform\Core\Exception\RuntimeException` exception will be thrown but this value can easily be increased with a
bit of configuration:

```yaml
# config/packages/api_platform.yaml
api_platform:
    eager_loading:
        max_joins: 100
```

Be careful when you exceed this limit, it's often caused by the result of a circular reference. [Serializer groups](serialization.md)
can be a good solution to fix this issue.

#### Fetch Partial

If you want to fetch only partial data according to serialization groups, you can enable `fetch_partial` parameter:

```yaml
# config/packages/api_platform.yaml
api_platform:
    eager_loading:
        fetch_partial: true
```

It is disabled by default.
If enabled, Doctrine ORM entities will not work as expected if any of the other fields are used.

#### Force Eager

As mentioned above, by default we force eager loading for all relations. This behaviour can be modified in the
configuration in order to apply it only on join relations having the `EAGER` fetch mode:

```yaml
# config/packages/api_platform.yaml
api_platform:
    eager_loading:
        force_eager: false
```

#### Override at Resource and Operation Level

When eager loading is enabled, whatever the status of the `force_eager` parameter, you can easily override it directly
from the configuration of each resource. You can do this at the resource level, at the operation level, or both:

```php
<?php
// api/src/Entity/Address.php

namespace App\Entity;

use ApiPlatform\Core\Annotation\ApiResource;
use Doctrine\ORM\Mapping as ORM;

#[ORM\Entity]
#[ApiResource]
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

#[ORM\Entity]
#[ApiResource(forceEager: false)]
class User
{
    #[ORM\ManyToOne(fetch: 'EAGER')]
    public Address $address;

    /**
     * @var Group[]
     */
    #[ORM\ManyToMany(targetEntity: 'Group', inversedBy: 'users')]
    #[ORM\JoinTable(name: 'users_groups')]
    public $groups;

    // ...
}
```

```php
<?php
// api/src/Entity/Group.php

namespace App\Entity;

use ApiPlatform\Core\Annotation\ApiResource;
use Doctrine\ORM\Mapping as ORM;

#[ORM\Entity]
#[ApiResource(
    forceEager: false,
    itemOperations: [
        "get" => ["force_eager" => true],
        "post",
    ],
    collectionOperations; [
        "get" => ["force_eager" => true],
        "post",
    ]
)]
class Group
{
    /**
     * @var User[]
     */
    #[ORM\ManyToMany(targetEntity: 'User', mappedBy: 'groups')] 
    public $users;

    // ...
}
```

Be careful, the operation level is higher priority than the resource level but both are higher priority than the global
configuration.

#### Disable Eager Loading

If for any reason you don't want the eager loading feature, you can turn it off in the configuration:

```yaml
# config/packages/api_platform.yaml
api_platform:
    eager_loading:
        enabled: false
```

The whole configuration described before will no longer work and Doctrine will recover its default behavior.

### Partial Pagination

When using the default pagination, the Doctrine paginator will execute a `COUNT` query on the collection. The result of the
`COUNT` query is used to compute the last page available. With big collections this can lead to quite long response times.
If you don't mind not having the last page available, you can enable partial pagination and avoid the `COUNT` query:

```yaml
# config/packages/api_platform.yaml
api_platform:
    collection:
        pagination:
            partial: true # Disabled by default
```

More details are available on the [pagination documentation](pagination.md#partial-pagination).

## Profiling with Blackfire.io

Blackfire.io allows you to monitor the performance of your applications. For more information, visit the [Blackfire.io website](https://blackfire.io/).

To configure Blackfire.io follow these simple steps:

1. Add the following to your `docker-compose.override.yml` file:

    ```yaml
        blackfire:
            image: blackfire/blackfire:2
            environment:
                # Exposes the host BLACKFIRE_SERVER_ID and TOKEN environment variables.
                - BLACKFIRE_SERVER_ID
                - BLACKFIRE_SERVER_TOKEN
                - BLACKFIRE_DISABLE_LEGACY_PORT=1
    ```

2. Add your Blackfire.io id and server token to your `.env` file at the root of your project (be sure not to commit this to a public repository):

    ```shell
    BLACKFIRE_SERVER_ID=xxxxxxxxxx
    BLACKFIRE_SERVER_TOKEN=xxxxxxxxxx
    ```

    Or set it in the console before running Docker commands:

    ```shell
    export BLACKFIRE_SERVER_ID=xxxxxxxxxx
    export BLACKFIRE_SERVER_TOKEN=xxxxxxxxxx
    ```

3. Install and configure the Blackfire probe in the app container, by adding the following to your `./Dockerfile`:

    ```dockerfile
            RUN version=$(php -r "echo PHP_MAJOR_VERSION.PHP_MINOR_VERSION;") \
                && curl -A "Docker" -o /tmp/blackfire-probe.tar.gz -D - -L -s https://blackfire.io/api/v1/releases/probe/php/alpine/amd64/$version \
                && mkdir -p /tmp/blackfire \
                && tar zxpf /tmp/blackfire-probe.tar.gz -C /tmp/blackfire \                        
                && mv /tmp/blackfire/blackfire-*.so $(php -r "echo ini_get('extension_dir');")/blackfire.so \
                && printf "extension=blackfire.so\nblackfire.agent_socket=tcp://blackfire:8307\n" > $PHP_INI_DIR/conf.d/blackfire.ini
    ```

4. Rebuild and restart all your containers

    ```console
    docker-compose build
    docker-compose up -d
    ```

For details on how to perform profiling, see [the Blackfire.io documentation](https://blackfire.io/docs/integrations/docker#using-the-client-for-http-profiling).

# Performance and Cache

## Enabling the Built-in HTTP Cache Invalidation System

Exposing a hypermedia API has [many advantages](http://blog.theamazingrando.com/in-band-vs-out-of-band.html). One
is the ability to know exactly which resources are included in HTTP responses created by the API. We used this specificity
to make API Platform apps blazing fast.

When the cache mechanism [is enabled](configuration.md), API Platform collects identifiers of every resource included in
a given HTTP response (including lists, embedded documents, and subresources) and returns them in a special HTTP header
called [Cache-Tags](https://support.cloudflare.com/hc/en-us/articles/206596608-How-to-Purge-Cache-Using-Cache-Tags-Enterprise-only-).

A caching [reverse proxy](https://en.wikipedia.org/wiki/Reverse_proxy) supporting cache tags (e.g. Varnish, Cloudflare,
Fastly) must be put in front of the web server and store all responses returned by the API with a high [TTL](https://en.wikipedia.org/wiki/Time_to_live).
This means that after the first request, all subsequent requests will not hit the web server, and will be served instantly
from the cache.

When a resource is modified, API Platform takes care of purging all responses containing it in the proxy’s
cache. This ensures that the content served will always be fresh because the cache is purged in real time. Support for
most specific cases such as the invalidation of collections when a document is added or removed or for relationships and
inverse relations is built-in.

### Integrations

#### Built-in Caddy HTTP Cache

The API Platform distribution relies on the [Caddy web server](https://caddyserver.com) which provides an official HTTP cache module called [cache-handler](https://github.com/caddyserver/cache-handler), that is based on [Souin](https://github.com/darkweak/souin).

The integration using the cache handler is quite simple. You just have to update the `api/Dockerfile` to build your caddy instance with the HTTP cache

```diff
# Versions
-FROM dunglas/frankenphp:1-php8.3 AS frankenphp_upstream

+FROM dunglas/frankenphp:1-builder-php8.3 AS builder
+COPY --from=caddy:builder /usr/bin/xcaddy /usr/bin/xcaddy
+
+RUN apt-get update && apt-get install --no-install-recommends -y \
+    git
+
+ENV CGO_ENABLED=1 XCADDY_SETCAP=1 XCADDY_GO_BUILD_FLAGS="-ldflags \"-w -s -extldflags '-Wl,-z,stack-size=0x80000'\""
+RUN xcaddy build \
+    --output /usr/local/bin/frankenphp \
+    --with github.com/dunglas/frankenphp/caddy \
+    --with github.com/dunglas/mercure/caddy \
+    --with github.com/dunglas/vulcain/caddy \
+    --with github.com/dunglas/caddy-cbrotli \
+    # You should use another storage than the default one (e.g. otter).
+    # The list of the available storages can be find either on the documentation website (https://docs.souin.io/docs/storages/) or on the storages repository https://github.com/darkweak/storages
+    --with github.com/caddyserver/cache-handler
+    # Or use the following lines instead of the cache-handler one for the latest improvements
+    #--with github.com/darkweak/souin/plugins/caddy \
+    #--with github.com/darkweak/storages/otter/caddy
+
+FROM dunglas/frankenphp:latest AS frankenphp_upstream
+COPY --from=builder --link /usr/local/bin/frankenphp /usr/local/bin/frankenphp
```

Update your Caddyfile with the following configuration:

```caddyfile
{
    cache
    # ...
}

# ...
```

This will tell to caddy to use the HTTP cache and activate the tag-based invalidation API. You can refer to the [cache-handler documentation](https://github.com/caddyserver/cache-handler) or the [souin website documentation](https://docs.souin.io) to learn how to configure the HTTP cache server.

Set up HTTP cache invalidation in your API Platform project using the Symfony or Laravel configuration below:

##### Cache Invalidation Configuration using Symfony

```yaml
api_platform:
  http_cache:
    invalidation:
      # We assume that your API can reach your caddy instance by the hostname http://caddy.
      # The endpoint /souin-api/souin is the default path to the invalidation API.
      urls: ['http://caddy/souin-api/souin']
      purger: api_platform.http_cache.purger.souin
```

##### Cache Invalidation Configuration using Laravel

```php
<?php
// config/api-platform.php
return [
    // ....
    'http_cache' => [
        'invalidation' => [
            // We assume that your API can reach your caddy instance by the hostname http://caddy.
            // The endpoint /souin-api/souin is the default path to the invalidation API.
            'urls' => ['http://caddy/souin-api/souin'],
            'purger' => 'api_platform.http_cache.purger.souin',
        ]
    ],
];
```

Don't forget to set your `Cache-Control` directive to enable caching on your API resource class.
This can be achieved using the `cacheHeaders` property:

```php
use ApiPlatform\Metadata\ApiResource;

#[ApiResource(
    cacheHeaders: [
        'public' => true,
        'max_age' => 60,
    ]
)]
class Book
{
    // ...
}
```

And voilà, you have a fully working HTTP cache with an invalidation API.

#### Varnish

Integration with Varnish and Doctrine ORM is shipped with the core library.

##### Varnish cache invalidation system using Symfony

Add the following configuration to enable the cache invalidation system:

```yaml
api_platform:
  http_cache:
    invalidation:
      enabled: true
      varnish_urls: ['%env(VARNISH_URL)%']
  defaults:
    cache_headers:
      max_age: 0
      shared_max_age: 3600
      vary: ['Content-Type', 'Authorization', 'Origin']
```

##### Varnish cache invalidation system using Laravel

Add the following configuration to enable the cache invalidation system:

```php
<?php
// config/api-platform.php
return [
    // ....
    'http_cache' => [
        'invalidation' => [
            'enabled' => true,
            'varnish_urls' => ['%env(VARNISH_URL)%'],
        ]
    ],
    'defaults' => [
        'cache_headers' => [
            'max_age' => 0,
            'shared_max_age' => 3600,
            'vary' => ['Content-Type', 'Authorization', 'Origin'],
        ]
    ],
];
```

## Configuration

Support for reverse proxies other than Varnish or Caddy with the HTTP cache module can be added by implementing the `ApiPlatform\HttpCache\PurgerInterface`.
Three purgers are available, the built-in caddy HTTP cache purger (`api_platform.http_cache.purger.souin`), the HTTP tags (`api_platform.http_cache.purger.varnish.ban`), the surrogate key implementation
(`api_platform.http_cache.purger.varnish.xkey`). You can specify the implementation using the `purger` configuration node,
for example, to use the `Xkey` implementation see the Symfony or Laravel configuration below:

### Exemple of Varnish Xkey implementation using Symfony

```yaml
api_platform:
  http_cache:
    invalidation:
      enabled: true
      varnish_urls: ['%env(VARNISH_URL)%']
      purger: 'api_platform.http_cache.purger.varnish.xkey'
    public: true
  defaults:
    cache_headers:
      max_age: 0
      shared_max_age: 3600
      vary: ['Content-Type', 'Authorization', 'Origin']
      invalidation:
        xkey:
          glue: ', '
```

### Exemple of Varnish Xkey implementation using Laravel

```php
<?php
// config/api-platform.php
return [
    // ....
    'http_cache' => [
        'invalidation' => [
            'enabled' => true,
            'varnish_urls' => ['%env(VARNISH_URL)%'],
            'purger' => 'api_platform.http_cache.purger.varnish.xkey',
        ],
        'public' => true,
    ],
    'defaults' => [
        'cache_headers' => [
            'max_age' => 0,
            'shared_max_age' => 3600,
            'vary' => ['Content-Type', 'Authorization', 'Origin'],
            'invalidation' => [
                'xkey' => [
                    'glue' => ', ',
                ]
            ],
        ]
    ],
];
```


In addition to the cache invalidation mechanism, you may want to [use HTTP/2 Server Push to pre-emptively send relations
to the client](push-relations.md).

### Extending Cache-Tags for Invalidation

Sometimes you need individual resources like `/me`. To work properly, the `Cache-Tags` header needs to be
augmented with these resources. Here is an example of how this can be done:

```php
<?php
// api/src/EventSubscriber/UserResourcesSubscriber.php
namespace App\EventSubscriber;

use ApiPlatform\Symfony\EventListener\EventPriorities;
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

The `cacheHeaders` attribute can be used to set custom HTTP cache headers:

```php
use ApiPlatform\Metadata\ApiResource;

#[ApiResource(
    cacheHeaders: [
        'max_age' => 60,
        'shared_max_age' => 120,
        'vary' => ['Authorization', 'Accept-Language'],
    ]
)]
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
use ApiPlatform\Metadata\ApiResource;
use ApiPlatform\Metadata\Get;

#[ApiResource]
#[Get(
    cacheHeaders: [
        'max_age' => 60,
        'shared_max_age' => 120,
    ]
)]
class Book
{
    // ...
}
```

## Enabling the Metadata Cache

Computing metadata used by the bundle is a costly operation. Fortunately, metadata can be computed once and then cached.
API Platform internally uses a [PSR-6](https://www.php-fig.org/psr/psr-6/) cache. If the Symfony Cache component is available
(the default in the API Platform distribution), it automatically enables support for the best cache adapter available.

Best performance is achieved using [APCu](https://github.com/krakjoe/apcu). Be sure to have the APCu extension installed
on your production server (this is the case by default in the Docker image provided by the API Platform distribution).
API Platform will automatically use it.

## Using FrankenPHP's Worker Mode

API response times can be significantly improved by enabling [FrankenPHP's worker mode](https://frankenphp.dev/docs/worker/).
This feature is enabled by default in the production environment of the API Platform distribution.

## Doctrine Queries and Index

### Search Filter

When using the `SearchFilter` and case insensitivity, Doctrine will use the `LOWER` SQL function. Depending on your
driver, you may want to carefully index it by using a [function-based index,](https://use-the-index-luke.com/sql/where-clause/functions/case-insensitive-search) or it will impact performance
with a huge collection. [Here are some examples to index LIKE filters](https://use-the-index-luke.com/sql/where-clause/searching-for-ranges/like-performance-tuning) depending on your database driver.

### Eager Loading

By default, Doctrine comes with [lazy loading](https://www.doctrine-project.org/projects/doctrine-orm/en/current/reference/working-with-objects.html#by-lazy-loading)
- usually a killer time-saving feature but also a performance killer with large applications.

Fortunately, Doctrine offers another approach to solve this problem: [eager loading](https://www.doctrine-project.org/projects/doctrine-orm/en/current/reference/working-with-objects.html#by-eager-loading).
This can easily be enabled for a relation: `#[ORM\ManyToOne(fetch: "EAGER")]`.

By default, in API Platform, we chose to force eager loading for all relations, with or without the Doctrine
`fetch` attribute. Thanks to the eager loading [extension](extensions.md). The `EagerLoadingExtension` will join every
readable association according to the serialization context. If you want to fetch an association that is not serializable,
you have to bypass `readable` and `readableLink` by using the `fetchEager` attribute on the property declaration, for example:

```php
...

#[ApiProperty(fetchEager: true)]
public $foo;

...
```

> **Warning**: to trigger the `EagerLoadingExtension` you must use [Serializer groups](serialization.md) on relations properties.

#### Max Joins

There is a default restriction with this feature. We allow up to 30 joins per query. Beyond that, an
`ApiPlatform\Exception\RuntimeException` exception will be thrown but this value can easily be increased with a
bit of configuration:

```yaml
# api/config/packages/api_platform.yaml
api_platform:
  eager_loading:
    max_joins: 100
```

Be careful when you exceed this limit, it's often caused by the result of a circular reference. [Serializer groups](serialization.md)
can be a good solution to fix this issue.

#### Fetch Partial

If you want to fetch only partial data according to serialization groups, you can enable the `fetch_partial` parameter:

```yaml
# api/config/packages/api_platform.yaml
api_platform:
  eager_loading:
    fetch_partial: true
```

It is disabled by default.
If enabled, Doctrine ORM entities will not work as expected if any of the other fields are used.

#### Force Eager

As mentioned above, by default we force eager loading for all relations. This behavior can be modified in the
configuration to apply it only on join relations having the `EAGER` fetch mode:

```yaml
# api/config/packages/api_platform.yaml
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

use ApiPlatform\Metadata\ApiResource;
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

use ApiPlatform\Metadata\ApiResource;
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
    #[ORM\ManyToMany(targetEntity: Group::class, inversedBy: 'users')]
    #[ORM\JoinTable(name: 'users_groups')]
    public $groups;

    // ...
}
```

```php
<?php
// api/src/Entity/Group.php
namespace App\Entity;

use ApiPlatform\Metadata\ApiResource;
use ApiPlatform\Metadata\Get;
use ApiPlatform\Metadata\Post;
use ApiPlatform\Metadata\GetCollection;
use Doctrine\ORM\Mapping as ORM;

#[ApiResource(forceEager: false)]
#[Get(forceEager: true)]
#[Post]
#[GetCollection(forceEager: true)]
#[ORM\Entity]
class Group
{
    /**
     * @var User[]
     */
    #[ORM\ManyToMany(targetEntity: User::class, mappedBy: 'groups')]
    public $users;

    // ...
}
```

Be careful, the operation level has a higher priority than the resource level but both are higher priority than the global
configuration.

#### Disable Eager Loading

If for any reason you don't want the eager loading feature, you can turn it off in the configuration:

```yaml
# api/config/packages/api_platform.yaml
api_platform:
  eager_loading:
    enabled: false
```

The whole configuration described before will no longer work and Doctrine will recover its default behavior.

### Partial Pagination

When using the default pagination, the Doctrine paginator will execute a `COUNT` query on the collection. The result of the
`COUNT` query is used to compute the last page available. With big collections, this can lead to quite long response times.
If you don't mind not having the last page available, you can enable partial pagination and avoid the `COUNT` query:

```yaml
# api/config/packages/api_platform.yaml
api_platform:
  defaults:
    pagination_partial: true # Disabled by default
```

More details are available on the [pagination documentation](pagination.md#partial-pagination).

## Profiling with Blackfire.io

Blackfire.io allows you to monitor the performance of your applications. For more information, visit the [Blackfire.io website](https://blackfire.io/).

To configure Blackfire.io follow these steps:

1. Add the following to your `compose.override.yaml` file:

```yaml
services:
  # ...
  blackfire:
    image: blackfire/blackfire:2
    environment:
      # Exposes the host BLACKFIRE_SERVER_ID and TOKEN environment variables.
      - BLACKFIRE_SERVER_ID
      - BLACKFIRE_SERVER_TOKEN
      - BLACKFIRE_DISABLE_LEGACY_PORT=1
```

2. Add your Blackfire.io ID and server token to your `.env` file at the root of your project (be sure not to commit this to a public repository):

```console
BLACKFIRE_SERVER_ID=xxxxxxxxxx
BLACKFIRE_SERVER_TOKEN=xxxxxxxxxx
```

Or set it in the console before running Docker commands:

```console
export BLACKFIRE_SERVER_ID=xxxxxxxxxx
export BLACKFIRE_SERVER_TOKEN=xxxxxxxxxx
```

3. Install and configure the Blackfire probe in the app container, by adding the following to your `./Dockerfile`:

```dockerfile
        RUN version=$(php -r "echo PHP_MAJOR_VERSION.PHP_MINOR_VERSION.(PHP_ZTS ? '-zts' : '');") \
            && architecture=$(uname -m) \
            && curl -A "Docker" -o /tmp/blackfire-probe.tar.gz -D - -L -s https://blackfire.io/api/v1/releases/probe/php/linux/$architecture/$version \
            && mkdir -p /tmp/blackfire \
            && tar zxpf /tmp/blackfire-probe.tar.gz -C /tmp/blackfire \
            && mv /tmp/blackfire/blackfire-*.so $(php -r "echo ini_get ('extension_dir');")/blackfire.so \
            && printf "extension=blackfire.so\nblackfire.agent_socket=tcp://blackfire:8307\n" > $PHP_INI_DIR/conf.d/blackfire.ini \
            && rm -rf /tmp/blackfire /tmp/blackfire-probe.tar.gz
```

4. Rebuild and restart all your containers

```console
docker compose build
docker compose up --wait
```

For details on how to perform profiling, see [the Blackfire.io documentation](https://blackfire.io/docs/integrations/docker#using-the-client-for-http-profiling).

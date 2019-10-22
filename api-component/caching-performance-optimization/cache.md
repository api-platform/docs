# HTTP Cache

## Enabling the Built-in HTTP Cache Invalidation System

Exposing a hypermedia API has [many advantages](http://blog.theamazingrando.com/in-band-vs-out-of-band.html). One of them
is the ability to know exactly which resources are included in HTTP responses created by the API. We used this specificity
to make API Platform apps blazing fast.

When the cache mechanism [is enabled](../usage-and-configuration/configuration.md), API Platform collects identifiers of every resource included in
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
included in the Docker setup provided with the [API Platform distribution](../../distribution/index.md). If you use the distribution,
this feature works out of the box.

If you don't use the distribution, add the following configuration to enable the cache invalidation system:

```yaml
api_platform:
    http_cache:
        invalidation:
            enabled: true
            varnish_urls: ['%env(VARNISH_URL)%']
        max_age: 0
        shared_max_age: 3600
        vary: ['Content-Type', 'Authorization', 'Origin']
        public: true
```

Support for reverse proxies other than Varnish can easily be added by implementing the `ApiPlatform\Core\HttpCache\PurgerInterface`.

In addition to the cache invalidation mechanism, you may want to [use HTTP/2 Server Push to pre-emptively send relations
to the client](../usage-and-configuration/push-relations.md).

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

## Setting Custom HTTP Cache Headers

The `cache_headers` attribute can be used to set custom HTTP cache headers:

```php
use ApiPlatform\Core\Annotation\ApiResource;

/**
 * @ApiResource(cacheHeaders={"max_age"=60, "shared_max_age"=120, "vary"={"Authorization", "Accept-Language"}})
 */
class Book
{
    // ...
}
```

For all endpoints related to this resource class, the following HTTP headers will be set:

```
Cache-Control: max-age=60, public, s-maxage=120
Vary: Authorization, Accept-Language
```

It's also possible to set different cache headers per operation:

```php
use ApiPlatform\Core\Annotation\ApiResource;

/**
 * @ApiResource(
 *     itemOperations={
 *         "get"={"cache_headers"={"max_age"=60, "shared_max_age"=120}}
 *     }
 * )
 */
class Book
{
    // ...
}
```

# Pushing Related Resources Using HTTP/2

> HTTP/2 allows a server to pre-emptively send (or "push") responses (along with corresponding "promised" requests) to
> a client in association with a previous client-initiated request. This can be useful when the server knows the client
> will need to have those responses available in order to fully process the response to the original request.
>
> —[RFC 7540](https://tools.ietf.org/html/rfc7540#section-8.2)

API Platform leverages this capability by pushing relations of a resource to clients.

> [!NOTE]
> We strongly recommend using [Vulcain](https://vulcain.rocks) instead of this feature.
> Vulcain is faster, cleaner, more flexible, and is supported out of the box in [the API Platform distribution](../symfony/index.md).

```php
<?php
// api/src/ApiResource/Book.php with Symfony or app/ApiResource/Book.php with Laravel
namespace App\ApiResource;

use ApiPlatform\Metadata\ApiProperty;
use ApiPlatform\Metadata\ApiResource;

#[ApiResource]
class Book
{
     #[ApiProperty(push: true)]
    public Author $author;

    // ...
}
```

By setting the `push` attribute to `true` on a property holding a relation, API Platform will automatically add a valid
`Link` HTTP header with the `preload` relation. According to the [Preload W3C Candidate Recommendation](https://www.w3.org/TR/preload/#server-push-http-2),
web servers and proxy servers can read this header, fetch the related resource and send it to the client using Server Push.

With the Caddy web server (the server shipped as part of the [distribution](../symfony/index.md)), you must add
[the `push` directive](https://caddyserver.com/docs/caddyfile/directives/push) to your `Caddyfile` to be able to use this feature.

[NGINX](https://www.nginx.com/blog/nginx-1-13-9-http2-server-push/), [Apache](https://httpd.apache.org/docs/current/howto/http2.html#push),
[Cloudflare](https://www.cloudflare.com/website-optimization/http2/serverpush/), [Fastly](https://docs.fastly.com/guides/performance-tuning/http2-server-push)
and [Akamai](https://blogs.akamai.com/2017/03/http2-server-push-the-what-how-and-why.html) honor this header.

Using this feature maximises HTTP cache hits for your API resources.
For best performance, this feature should be used in conjunction with [the built-in HTTP cache invalidation system (based on Varnish)](performance.md#enabling-the-built-in-http-cache-invalidation-system).

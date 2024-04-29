# Configuring the Caddy Web Server

[The API Platform distribution](index.md) is shipped with [the Caddy web server](https://caddyserver.com).
The build contains the [Mercure](../core/mercure.md) and the [Vulcain](https://vulcain.rocks) Caddy modules.

Caddy is positioned in front of the web API and of the Progressive Web App.
It routes requests to either service depending on the value of the `Accept` HTTP header or the extension
of the requested file.

Using the same domain to serve the API and the PWA [improves performance by preventing unnecessary CORS preflight requests
and encourages embracing the REST principles](https://dunglas.fr/2022/01/preventing-cors-preflight-requests-using-content-negotiation/).

By default, requests having an `Accept` request header containing the `text/html` media type are routed to the Next.js application,
except for some paths known to be resources served by the API (e.g. the Swagger UI documentation, static files provided by bundles...).
Other requests are routed to the API.

Sometimes, you may want to let the PHP application generate HTML responses.
For instance, when you create your own Symfony controllers serving HTML pages,
or when using bundles such as EasyAdmin or SonataAdmin.

To do so, you have to tweak the rules used to route the requests.
Open `api-platform/api/frankenphp/Caddyfile` and modify the expression.
You can use [any CEL (Common Expression Language) expression](https://caddyserver.com/docs/caddyfile/matchers#expression) supported by Caddy.

For instance, if you want to route all requests to a path starting with `/admin` to the API, modify the existing expression like this:

```patch
# Matches requests for HTML documents, for static files and for Next.js files,
# except for known API paths and paths with extensions handled by API Platform
@pwa expression `(
        {header.Accept}.matches("\\btext/html\\b")
-        && !{path}.matches("(?i)(?:^/docs|^/graphql|^/bundles/|^/_profiler|^/_wdt|\\.(?:json|html$|csv$|ya?ml$|xml$))")
+        && !{path}.matches("(?i)(?:^/admin|^/docs|^/graphql|^/bundles/|^/_profiler|^/_wdt|\\.(?:json|html$|csv$|ya?ml$|xml$))")
    )`
```

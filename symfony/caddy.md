# Configuring the Caddy Web Server with Symfony

[The API Platform distribution](index.md) is shipped with [the Caddy web server](https://caddyserver.com).
The build contains the [Mercure](../core/mercure.md) and the [Vulcain](https://vulcain.rocks) Caddy modules.

Caddy is positioned in front of the web API and of the Progressive Web App (PWA).
It routes requests to either service depending on the value of the `Accept` HTTP header or the path of the request.

Using the same domain to serve the API and the PWA [improves performance by preventing unnecessary CORS preflight requests
and encourages embracing the REST principles](https://dunglas.fr/2022/01/preventing-cors-preflight-requests-using-content-negotiation/).

## Why `route {}` Is Required

Caddy processes directives in a [predefined global order](https://caddyserver.com/docs/caddyfile/directives#directive-order),
not in the order they appear in the Caddyfile. In that order, `rewrite` runs **before** `reverse_proxy`. Without
explicit ordering, a browser request to `/` would match the `@phpRoute` rewrite condition and be rewritten to
`index.php` before Caddy ever evaluated whether the request should be proxied to Next.js.

Wrapping the directives in a `route {}` block enforces **strict first-match-wins evaluation in file order**. The first
directive that matches a request wins, and Caddy stops evaluating the rest. This is what makes the `@pwa` proxy check
run before the PHP rewrite:

```caddy
route {
    # 1. Check @pwa first — proxy to Next.js if matched
    reverse_proxy @pwa http://{$PWA_UPSTREAM}

    # 2. Only if @pwa did not match, rewrite to index.php
    @phpRoute { not path /.well-known/mercure*; not file {path} }
    rewrite @phpRoute index.php

    # 3. Run PHP for index.php
    @frontController path index.php
    php @frontController

    # 4. Serve remaining static files
    file_server { hide *.php }
}
```

## The `@pwa` Matcher

The `@pwa` named matcher is a [CEL (Common Expression Language) expression](https://caddyserver.com/docs/caddyfile/matchers#expression)
that decides which requests are forwarded to the Next.js application:

```caddy
@pwa expression `(
        header({'Accept': '*text/html*'})
        && !path(
            '/docs*', '/graphql*', '/bundles*', '/contexts*', '/_profiler*', '/_wdt*',
            '*.json*', '*.html', '*.csv', '*.yml', '*.yaml', '*.xml'
        )
    )
    || path('/favicon.ico', '/manifest.json', '/robots.txt', '/sitemap*', '/_next*', '/__next*')
    || query({'_rsc': '*'})`
```

The expression has three independent clauses joined by `||`. A request matches `@pwa` if **any** clause is true.

### Clause 1: HTML requests that are not API paths

A browser navigating to any URL sends `Accept: text/html, */*`. This clause forwards those requests to Next.js unless
the path is known to be served by the API or carries an extension that API Platform handles through
[content negotiation](../core/content-negotiation.md).

Paths excluded from Next.js (handled by PHP instead):

| Pattern | Reason |
|---|---|
| `/docs*` | Swagger UI and OpenAPI documentation |
| `/graphql*` | GraphQL endpoint |
| `/bundles*` | Symfony bundle assets published by `assets:install` |
| `/contexts*` | JSON-LD context documents |
| `/_profiler*`, `/_wdt*` | Symfony Web Debug Toolbar and Profiler |
| `*.json*`, `*.html`, `*.csv`, `*.yml`, `*.yaml`, `*.xml` | Content-negotiated formats served by the API |

### Clause 2: Next.js static assets and well-known files

```
path('/favicon.ico', '/manifest.json', '/robots.txt', '/sitemap*', '/_next*', '/__next*')
```

These paths are forwarded to Next.js unconditionally, regardless of the `Accept` header. `/_next/*` and `/__next/*`
are the internal asset paths used by the Next.js runtime for JavaScript chunks, CSS, images, and hot module
replacement updates in development.

### Clause 3: React Server Components requests

```
query({'_rsc': '*'})
```

Next.js uses the `_rsc` query parameter internally for
[React Server Components](https://nextjs.org/docs/app/building-your-application/rendering/server-components) data
fetching. These requests do not carry `text/html` in their `Accept` header, so they would miss clause 1 without this
dedicated check.

## The `Link` Header

```caddy
header ?Link `</docs.jsonld>; rel="http://www.w3.org/ns/hydra/core#apiDocumentation", </.well-known/mercure>; rel="mercure"`
```

This directive is placed at the **site block level**, outside the `route {}` block, so it applies to every response
regardless of whether it came from PHP or Next.js. The `?` prefix means the header is only set when not already
present in the response — a PHP response that sets its own `Link` header is not overwritten.

Setting this at the Caddy level serves two purposes:

1. **API discoverability**: every response advertises the Hydra API documentation URL, allowing clients to
   auto-discover the API.
2. **Mercure subscription**: every response advertises the Mercure hub URL, so clients can subscribe to real-time
   updates without any application code.

The Next.js application does not need to set these headers itself — they arrive on every response automatically.

## The `PWA_UPSTREAM` Environment Variable

```caddy
reverse_proxy @pwa http://{$PWA_UPSTREAM}
```

`PWA_UPSTREAM` is resolved at runtime from the container environment. In `compose.yaml` it is set to `pwa:3000`,
where `pwa` is the Docker Compose service name and `3000` is the default port of the Next.js server.

When the `pwa` service is not running (for example in an API-only project), Caddy returns a `502 Bad Gateway` for any
request matching `@pwa`. To run without a Next.js frontend, comment out that line in the Caddyfile:

```caddy
route {
    # Comment the following line if you don't want Next.js to catch requests for HTML documents.
    # In this case, they will be handled by the PHP app.
    # reverse_proxy @pwa http://{$PWA_UPSTREAM}

    @phpRoute { not path /.well-known/mercure*; not file {path} }
    rewrite @phpRoute index.php
    @frontController path index.php
    php @frontController
    file_server { hide *.php }
}
```

## Adjusting the Routing Rules

### Routing an admin path to PHP

If you use EasyAdmin, SonataAdmin, or a custom Symfony controller that serves HTML pages, add the path prefix to the
exclusion list inside clause 1 so those requests bypass Next.js:

```caddy
@pwa expression `(
        header({'Accept': '*text/html*'})
        && !path(
            '/admin*',
            '/docs*', '/graphql*', '/bundles*', '/contexts*', '/_profiler*', '/_wdt*',
            '*.json*', '*.html', '*.csv', '*.yml', '*.yaml', '*.xml'
        )
    )
    || path('/favicon.ico', '/manifest.json', '/robots.txt', '/sitemap*', '/_next*', '/__next*')
    || query({'_rsc': '*'})`
```

You can use [any CEL expression](https://caddyserver.com/docs/caddyfile/matchers#expression) supported by Caddy.

### Adding a custom API prefix

If your API is mounted under a prefix such as `/api`, add it to the exclusion list:

```caddy
&& !path(
    '/api*',
    '/docs*', '/graphql*', ...
)
```

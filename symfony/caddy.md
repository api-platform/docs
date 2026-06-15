# Configuring the Caddy Web Server with Symfony

When you scaffold a project with the [API Platform CLI](index.md), the Symfony application is built
on top of [`symfony-docker`](https://github.com/dunglas/symfony-docker), which ships
[the Caddy web server](https://caddyserver.com) running [FrankenPHP](https://frankenphp.dev). The
build contains the [Mercure](../core/mercure.md) and the [Vulcain](https://vulcain.rocks) Caddy
modules.

The Caddyfile lives at `api/frankenphp/Caddyfile`.

## How the CLI Serves the API and the PWA

By default the API and the Progressive Web App (PWA) are served **separately**:

- Caddy serves the **API** on `https://localhost`.
- When you scaffold with `--with-pwa`, the Next.js application runs **standalone** with `pnpm dev`
  on `http://localhost:3000`. It calls the API cross-origin using the `NEXT_PUBLIC_API_ENTRYPOINT`
  value written to `pwa/.env.local`, and the CLI installs and configures
  [`nelmio/cors-bundle`](https://github.com/nelmio/NelmioCorsBundle) on the API so those
  cross-origin requests are allowed.

This keeps the two applications independent and requires no Caddy configuration. If you prefer to
serve both on the **same domain** through Caddy — which
[improves performance by preventing unnecessary CORS preflight requests and encourages embracing the REST principles](https://dunglas.fr/2022/01/preventing-cors-preflight-requests-using-content-negotiation/)
— see
[Serving the API and the PWA on the Same Domain](#serving-the-api-and-the-pwa-on-the-same-domain)
below.

## The Shipped Caddyfile

Out of the box, `api/frankenphp/Caddyfile` routes every request to the PHP application. The relevant
part of the site block looks like this:

```caddy
{$SERVER_NAME:localhost} {
    root /app/public
    encode zstd br gzip

    mercure {
        # ...Mercure hub configuration...
    }

    vulcain

    # Extra directives injected by the CLI (see "The Link Header" below)
    {$CADDY_SERVER_EXTRA_DIRECTIVES}

    @phpRoute {
        not path /.well-known/mercure*
        not file {path}
    }
    rewrite @phpRoute index.php

    @frontController path index.php
    php @frontController {
        worker {
            file ./public/index.php
        }
    }

    file_server {
        hide *.php
    }
}
```

Any request that is not an existing static file and is not a Mercure subscription is rewritten to
`index.php` and handled by Symfony through the FrankenPHP worker.

## The `Link` Header

The CLI adds a Hydra + Mercure `Link` header to every response. Rather than editing the Caddyfile
directly, it injects the directive through the `CADDY_SERVER_EXTRA_DIRECTIVES` environment variable
in `api/compose.yaml`, inside a recipe block:

```yaml
###> api-platform/api-platform ###
CADDY_SERVER_EXTRA_DIRECTIVES:
    'header ?Link `</docs.jsonld>; rel="http://www.w3.org/ns/hydra/core#apiDocumentation",
    </.well-known/mercure>; rel="mercure"`'
###< api-platform/api-platform ###
```

The `?` prefix means the header is only set when not already present in the response — a PHP
response that sets its own `Link` header is not overwritten.

Setting it at the Caddy level serves two purposes:

1. **API discoverability**: every response advertises the Hydra API documentation URL, allowing
   clients to auto-discover the API.
2. **Mercure subscription**: every response advertises the Mercure hub URL, so clients can subscribe
   to real-time updates without any application code.

## Serving the API and the PWA on the Same Domain

If you want Caddy to serve both the API and the Next.js application on a single domain, you need to
forward HTML requests to the PWA and keep API requests on PHP. This is **not** configured by the
CLI; the steps below add it on top of a scaffolded project.

### 1. Make the PWA reachable from the Caddy container

Caddy runs inside the `php` container, so it must be able to reach the Next.js server. Either:

- run the PWA in a Docker service named `pwa` listening on port `3000` (then the upstream is
  `pwa:3000`), or
- keep running `pnpm dev` on the host and target it with `host.docker.internal:3000`.

Declare the upstream as an environment variable for the `php` service in `api/compose.yaml`:

```yaml
services:
    php:
        environment:
            PWA_UPSTREAM: pwa:3000
```

### 2. Wrap the routing directives in a `route {}` block

Caddy processes directives in a
[predefined global order](https://caddyserver.com/docs/caddyfile/directives#directive-order), not in
the order they appear in the Caddyfile. In that order, `rewrite` runs **before** `reverse_proxy`.
Without explicit ordering, a browser request to `/` would match the `@phpRoute` rewrite condition
and be rewritten to `index.php` before Caddy ever evaluated whether the request should be proxied to
Next.js.

Wrapping the directives in a `route {}` block enforces **strict first-match-wins evaluation in file
order**. The first directive that matches a request wins, and Caddy stops evaluating the rest. This
makes the `@pwa` proxy check run before the PHP rewrite. Replace the `@phpRoute … file_server`
section of the site block with:

```caddy
route {
    # 1. Check @pwa first — proxy to Next.js if matched
    reverse_proxy @pwa http://{$PWA_UPSTREAM}

    # 2. Only if @pwa did not match, rewrite to index.php
    @phpRoute { not path /.well-known/mercure*; not file {path} }
    rewrite @phpRoute index.php

    # 3. Run PHP for index.php
    @frontController path index.php
    php @frontController {
        worker {
            file ./public/index.php
        }
    }

    # 4. Serve remaining static files
    file_server { hide *.php }
}
```

### 3. Define the `@pwa` matcher

Add a `@pwa` named matcher — a
[CEL (Common Expression Language) expression](https://caddyserver.com/docs/caddyfile/matchers#expression)
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

The expression has three independent clauses joined by `||`. A request matches `@pwa` if **any**
clause is true.

#### Clause 1: HTML requests that are not API paths

A browser navigating to any URL sends `Accept: text/html, */*`. This clause forwards those requests
to Next.js unless the path is known to be served by the API or carries an extension that API
Platform handles through [content negotiation](../core/content-negotiation.md).

Paths excluded from Next.js (handled by PHP instead):

| Pattern                                                  | Reason                                              |
| -------------------------------------------------------- | --------------------------------------------------- |
| `/docs*`                                                 | Swagger UI and OpenAPI documentation                |
| `/graphql*`                                              | GraphQL endpoint                                    |
| `/bundles*`                                              | Symfony bundle assets published by `assets:install` |
| `/contexts*`                                             | JSON-LD context documents                           |
| `/_profiler*`, `/_wdt*`                                  | Symfony Web Debug Toolbar and Profiler              |
| `*.json*`, `*.html`, `*.csv`, `*.yml`, `*.yaml`, `*.xml` | Content-negotiated formats served by the API        |

#### Clause 2: Next.js static assets and well-known files

```caddy
path('/favicon.ico', '/manifest.json', '/robots.txt', '/sitemap*', '/_next*', '/__next*')
```

These paths are forwarded to Next.js unconditionally, regardless of the `Accept` header. `/_next/*`
and `/__next/*` are the internal asset paths used by the Next.js runtime for JavaScript chunks, CSS,
images, and hot module replacement updates in development.

#### Clause 3: React Server Components requests

```caddy
query({'_rsc': '*'})
```

Next.js uses the `_rsc` query parameter internally for
[React Server Components](https://nextjs.org/docs/app/building-your-application/rendering/server-components)
data fetching. These requests do not carry `text/html` in their `Accept` header, so they would miss
clause 1 without this dedicated check.

When the PWA upstream is unreachable, Caddy returns a `502 Bad Gateway` for any request matching
`@pwa`. To temporarily fall back to PHP-rendered HTML, comment out the `reverse_proxy @pwa` line
inside the `route {}` block.

## Adjusting the Routing Rules

The rules below assume you have enabled single-domain serving and therefore have a `@pwa` matcher to
tweak.

### Routing an admin path to PHP

If you use EasyAdmin, SonataAdmin, or a custom Symfony controller that serves HTML pages, add the
path prefix to the exclusion list inside clause 1 so those requests bypass Next.js:

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

You can use [any CEL expression](https://caddyserver.com/docs/caddyfile/matchers#expression)
supported by Caddy.

### Adding a custom API prefix

If your API is mounted under a prefix such as `/api`, add it to the exclusion list:

```caddy
&& !path(
    '/api*',
    '/docs*', '/graphql*', ...
)
```

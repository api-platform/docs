# Deploying behind caddy proxy

This tutorial describes how to use [the official API Platform distribution](../distribution/index.md) behind a caddy proxy (at a docker environment), e.g. if you want to use many apps at one server or at your dev machine.

## Preconditions

* A running caddy server, listen to ports 443 and 80: our proxy
* An API Platform app using the [API Platform distribution](../distribution/index.md)

In the tutorial I use the expression ''caddy proxy'' for the proxy itself and ''app caddy'' for the caddy server, that is used in the docker-compose.yaml of API Platform itself.

## Configuration

First, please remove all exposed ports from the app caddy service:

```diff
# docker-compose.yaml
...
  caddy:
    image: ${IMAGES_PREFIX:-}app-caddy
    depends_on:
      - php
      - pwa
    environment:
      PWA_UPSTREAM: pwa:3000
      SERVER_NAME: ${SERVER_NAME:-localhost}, caddy:80
      MERCURE_PUBLISHER_JWT_KEY: ${CADDY_MERCURE_JWT_SECRET:-!ChangeThisMercureHubJWTSecretKey!}
      MERCURE_SUBSCRIBER_JWT_KEY: ${CADDY_MERCURE_JWT_SECRET:-!ChangeThisMercureHubJWTSecretKey!}
    restart: unless-stopped
    volumes:
      - php_socket:/var/run/php
      - caddy_data:/data
      - caddy_config:/config
-   ports:
-     # HTTP
-     - target: 80
-       published: ${HTTP_PORT:-80}
-       protocol: tcp
-     # HTTPS
-     - target: 443
-       published: ${HTTPS_PORT:-443}
-       protocol: tcp
-     # HTTP/3
-     - target: 443
-       published: ${HTTP3_PORT:-443}
-       protocol: udp
...
```

These ports are used by your caddy proxy and not by the API Platform app caddy.

> All upcoming steps are shown in the main docker-compose.yaml.
>
> However, you probably want to implement them in docker-compose.overwrite.yaml or docker-compose.prod.yaml, depending on your environment.

### Adding networks

We introduce two docker networks for our services, one for the API Platform docker services to connect to each other (default), the other to connect to the caddy proxy server (proxy):

```yaml
# docker-compose.yaml (or docker-compose.overwrite.yaml / docker-compose.prod.yaml)

...

networks:
    proxy:
        external: true
    default:
```

The caddy proxy is attached to the external `proxy` network, too.

Now, each service in our `docker-compose.yaml` gets the following additional key: `networks`

```yaml
# docker-compose.yaml (or docker-compose.overwrite.yaml / docker-compose.prod.yaml)

...
services:
    php:
      networks:
       - default
...
```

Only the `caddy` service gets the additional `proxy` network:

```yaml
# docker-compose.yaml (or docker-compose.overwrite.yaml / docker-compose.prod.yaml)
...
services:

    caddy:
       networks:
        - default
        - proxy
...
```

### Configure caddy proxy

Now you can add a reverse entry to your caddy proxies `Caddyfile`. If you use the [Caddy docker proxy module](https://github.com/lucaslorentz/caddy-docker-proxy), you only have to add the following labels to your app caddy service:

```yaml
# docker-compose.yaml (or docker-compose.overwrite.yaml / docker-compose.prod.yaml)
...
services:

    caddy:
      labels:
        caddy: your-app.localhost
        caddy.reverse_proxy: "{{upstreams 80}}"
```

It will add the needed `Caddyfile` entry to your caddy proxies `Caddyfile` (which you can write directly, if you don't want to use the caddy docker proxy module):

```text
[Your App Domain] {
    reverse_proxy [app caddy container IP, proxy network]:80
}
```

e.g.

```text
your-app.localhost {
    reverse_proxy 172.19.0.2:80
}
```

Now you can reach your app with your domain name, but...

### Prevent redirect problems

Your API Platform application is configured to work behind a HTTPS connection and the app caddy service should handle the ssl connection.

In our setup the caddy proxy handles the ssl connection and internally redirects to an ''unsave'' port 80. This results in some problems that we have to solve:

Your app caddy will redirect you everytime from port 80 to port 443 (HTTPS), the browser connects to [https://your-app.localhost](https://your-app.localhost), caddy proxy redirects to 80, app caddy to 443 and so on.  

We can solve this by adding `http` in front of the server name (only) for the caddy app service. Then, caddy will not redirect to 443 internally:

```yaml
# docker-compose.yaml (or docker-compose.overwrite.yaml / docker-compose.prod.yaml)
...
services:
    caddy:
      environment:
        SERVER_NAME: http://${SERVER_NAME:-localhost}, caddy:80
...
```

Wohoo: If you open your domain, you can see the welcome page of API Platform, open the API page and the mercure debugger, ...

### Prevent CORS errors

... but you cannot open the admin application because a CORS error occurs.

The problem is that the caddy app service also works as a proxy for the PWA service, but this proxy runs with port 80 / http. We remember, all SSL connections are handled by the caddy proxy and not by our API Platform app.

The PWA sends a request to [https://your-app.localhost/](https://your-app.localhost/) with Content-Type `application/ld+json`. The caddy proxy redirects to the app caddy (http/port 80) which uses the php service to answer.
API Platform identifies the route, Content-Type and schema (which is HTTP at the moment) and sends a redirect to [http://your-app.localhost/docs.jsonld](http://your-app.localhost/docs.jsonld) to your PWA.
The PWA follows the redirect link and throw a CORS error because of the different schema (HTTP !== HTTPS).

To prevent this, we use a trick that is described in the symfony docs for [deployments behind a proxy](https://symfony.com/doc/current/deployment/proxies.html#custom-headers-when-using-a-reverse-proxy):

We command the proxy caddy to add a custom header with the original schema and replace the `HTTP_X_FORWARDED_PROTO` with this header at the `index.php`:

```php
<?php
// app/public/index.php

use App\Kernel;

// https://symfony.com/doc/current/deployment/proxies.html#custom-headers-when-using-a-reverse-proxy
$_SERVER['HTTP_X_FORWARDED_PROTO'] = $_SERVER['HTTP_CUSTOM_FORWARDED_PROTO'];

...
```

and

```yaml
# docker-compose.yaml (or docker-compose.overwrite.yaml / docker-compose.prod.yaml)
...
services:
    caddy:
        labels:
            caddy: your-app.localhost
            caddy.reverse_proxy: "{{upstreams 80}}"
            caddy.reverse_proxy.header_up: Custom-Forwarded-Proto {scheme}
...
```

This results in the following `Caddyfile` for the caddy proxy:

```text
your-app.localhost {
    reverse_proxy 172.19.0.2:80 {
        header_up Custom-Forwarded-Proto {scheme}
    }
}
```

Now, symfony generates the redirect to [https://your-app.localhost/docs.jsonld](https://your-app.localhost/docs.jsonld), and you can reach the API Platform admin without any CORS errors :-)

# Implement Træfik Into API Platform Dockerized

> An open-source reverse proxy and load balancer for HTTP and TCP-based applications that is easy, dynamic, automatic, fast, full-featured, production proven, provides metrics and integrates with every major cluster technology.
>
> —[https://traefik.io](https://traefik.io)

## Basic Implementation

This tutorial will help you to define your own routes for your client, api and more generally for your containers.

Use this custom API Platform `docker-compose.yml` file which implements ready-to-use Træfik container configuration. Override
ports and add labels to tell Træfik to listen on the routes mentioned and redirect routes to a specified container.

A few points to note:

* `--api.insecure=true` Tells Træfik to generate a browser view to watch containers and IP/DNS associated easier  
* `--providers.docker` Tells Træfik to listen on Docker Api  
* `labels:` Key for Træfik configuration into Docker integration

  ```yaml
  services:
  #  ...
    api:
      labels:
        - traefik.http.routers.api.rule=Host(`api.localhost`)
  ```

  The API DNS will be specified with ``traefik.http.routers.api.rule=Host(`your.host`)`` (here api.localhost)
* `--traefik.routers.clientloadbalancer.server.port=3000` The port specified to Træfik will be exposed by the container (here the React app exposes the 3000 port), but if your container exposes only one port, it can be ignored

We assume that you've generated a SSL `localhost.crt` and associated `localhost.key` combo under `./certs` folder
Then you edited your `admin/Dockerfile` and `client/Dockerfile` like this:

```diff
-ENV HTTPS true
+EXPOSE 3000
```

After that, don't forget to re-build your containers

```yaml
# docker-compose.yml
version: '3.4'

x-cache-from:
  - &cache
    cache_from:
      - ${NGINX_IMAGE:-quay.io/api-platform/nginx}
      - ${PHP_IMAGE:-quay.io/api-platform/php}

services:
  traefik:
      image: traefik:latest
      command: --api.insecure=true --providers.docker
      ports:
        - target: 80
          published: 80
          protocol: tcp
        - target: 443
          published: 443
          protocol: tcp
        - target: 8080
          published: 8080
          protocol: tcp
      volumes:
        - /var/run/docker.sock:/var/run/docker.sock

  php:
    build:
      context: ./api
      target: api_platform_php
      <<: *cache
    image: ${PHP_IMAGE:-quay.io/api-platform/php}
    healthcheck:
      interval: 10s
      timeout: 3s
      retries: 3
      start_period: 30s
    depends_on:
      - db
    volumes:
      - ./api:/srv/api:rw,cached
      - ./api/docker/php/conf.d/api-platform.dev.ini:/usr/local/etc/php/conf.d/api-platform.ini

  api:
    build:
      context: ./api
      target: api_platform_nginx
      <<: *cache
    image: ${NGINX_IMAGE:-quay.io/api-platform/nginx}
    depends_on:
      - php
    volumes:
      - ./api/public:/srv/api/public:ro

  vulcain:
    image: dunglas/vulcain
    environment:
      - CERT_FILE=/certs/localhost.crt
      - KEY_FILE=/certs/localhost.key
      - UPSTREAM=http://api
    depends_on:
      - api
    volumes:
      - ./certs:/certs:ro
    labels:
      - traefik.http.routers.vulcain.rule=Host(`vulcain.localhost`)

  db:
    image: postgres:12-alpine
    environment:
      - POSTGRES_DB=api
      - POSTGRES_PASSWORD=!ChangeMe!
      - POSTGRES_USER=api-platform
    volumes:
      - db-data:/var/lib/postgresql/data:rw
    labels:
      - traefik.http.routers.db.rule=Host(`db.localhost`)

  mercure:
    image: dunglas/mercure
    environment:
#      - ACME_HOSTS=${DOMAIN_NAME}
#      - CERT_FILE=/certs/localhost.crt
#      - KEY_FILE=/certs/localhost.key
      - JWT_KEY=${JWT_KEY}
      - ALLOW_ANONYMOUS=1
      - USE_FORWARDED_HEADERS=true
      - CORS_ALLOWED_ORIGINS=*
      - READ_TIMEOUT=0s
      - WRITE_TIMEOUT=0s
      - PUBLISH_ALLOWED_ORIGINS=*
    volumes:
      - ./certs:/certs:ro
    labels:
      - traefik.http.routers.mercure.rule=Host(`mercure.localhost`)

  client:
    build:
      context: ./client
      target: api_platform_client_development
      cache_from:
        - ${CLIENT_IMAGE:-quay.io/api-platform/client}
    image: ${CLIENT_IMAGE:-quay.io/api-platform/client}
    tty: true
    environment:
      - API_PLATFORM_CLIENT_GENERATOR_ENTRYPOINT=http://api
      - API_PLATFORM_CLIENT_GENERATOR_OUTPUT=src
    volumes:
      - ./client:/usr/src/client:rw,cached
    labels:
      - traefik.http.routers.client.rule=Host(`client.localhost`)
      - traefik.http.services.client.loadbalancer.server.port=3000

  admin:
    build:
      context: ./admin
      target: api_platform_admin_development
      cache_from:
        - ${ADMIN_IMAGE:-quay.io/api-platform/admin}
    image: ${ADMIN_IMAGE:-quay.io/api-platform/admin}
    tty: true
    volumes:
      - ./admin:/usr/src/admin:rw,cached
    labels:
      - traefik.http.routers.admin.rule=Host(`admin.localhost`)
      - traefik.http.services.admin.loadbalancer.server.port=3000

volumes:
  db-data: {}
```

Don't forget the db-data, or the database won't work in this dockerized solution.

`localhost` is a reserved domain referred to in your `/etc/hosts`.
If you want to implement custom DNS such as production DNS in local, just add them at the end of your `/etc/host` file like that:

```csv
# /etc/hosts
# ...

127.0.0.1       your.domain.com
```

If you do that, you'll have to update the `CORS_ALLOW_ORIGIN` environment variable `.env` to accept the specified URL.

## Known Issues

If your network is of type B, it may conflict with the Træfik sub-network.

## Going Further

As this Træfik configuration listens on 80 and 443 ports, you can run only 1 Træfik instance per server. However, you may want to run multiple API Platform projects on the same server. To deal with it, you'll have to externalize the Træfik configuration to another `docker-compose.yml` file, anywhere on your server.

Here is a working example:

```yaml
# /somewhere/docker-compose.yml
version: '3.4'

services:
  traefik:
    image: traefik:latest
    command: --api.insecure=true --providers.docker
    ports:
      - target: 80
        published: 80
        protocol: tcp
      - target: 443
        published: 443
        protocol: tcp
      - target: 8080
        published: 8080
        protocol: tcp
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
    networks:
      - api_platform_network
    # Add other networks here

networks:
  api_platform_network:
    external: true
  # Add other networks here
```

Then update the `docker-compose.yml` file belonging to your API Platform projects:

```diff
# /anywhere/api-platform/docker-compose.yml
version: '3.4'

x-cache:
  &cache
  cache_from:
    - ${CONTAINER_REGISTRY_BASE}/php
    - ${CONTAINER_REGISTRY_BASE}/nginx
    - ${CONTAINER_REGISTRY_BASE}/varnish

+x-network:
+  &network
+  networks:
+    - api_platform_network

services:
-  traefik:
-    image: traefik:latest
-    command: --api.insecure=true --providers.docker
-    ports:
-      - target: 80
-        published: 80
-        protocol: tcp
-      - target: 443
-        published: 443
-        protocol: tcp
-      - target: 8080
-        published: 8080
-        protocol: tcp
-    volumes:
-      - /var/run/docker.sock:/var/run/docker.sock

  php:
    build:
      context: ./api
      target: api_platform_php
      <<: *cache
    image: ${PHP_IMAGE:-quay.io/api-platform/php}
+    environment:
+      # You should remove these variables from .env into api folder
+      - TRUSTED_HOSTS=^(((${SUBDOMAINS_LIST}\.)?${DOMAIN_NAME})|api)$$
+      - CORS_ALLOW_ORIGIN=^${HTTP_OR_SSL}(${SUBDOMAINS_LIST}.)?${DOMAIN_NAME}$$
+      - DATABASE_URL=postgres://${DB_USER}:${DB_PASS}@db/${DB_NAME}
+      - MERCURE_SUBSCRIBE_URL=${HTTP_OR_SSL}mercure.${DOMAIN_NAME}$$
+      - MERCURE_PUBLISH_URL=${HTTP_OR_SSL}mercure.${DOMAIN_NAME}$$
+      - MERCURE_JWT_TOKEN=${JWT_KEY}
    healthcheck:
      interval: 10s
      timeout: 3s
      retries: 3
      start_period: 30s
    depends_on:
      - db
-      - dev-tls
    volumes:
      - ./api:/srv/api:rw,cached
      - ./api/docker/php/conf.d/api-platform.dev.ini:/usr/local/etc/php/conf.d/api-platform.ini
      - ./certs:/certs:ro
+    <<: *network

  api:
    build:
      context: ./api
      target: api_platform_nginx
      <<: *cache
    image: ${NGINX_IMAGE:-quay.io/api-platform/nginx}
    depends_on:
      - php
    volumes:
      - ./api/public:/srv/api/public:ro
    labels:
      - traefik.http.routers.api.rule=Host(`api.${DOMAIN_NAME}`)
+    <<: *network

  vulcain:
    image: dunglas/vulcain
    environment:
      - CERT_FILE=/certs/localhost.crt
      - KEY_FILE=/certs/localhost.key
      - UPSTREAM=http://api
    depends_on:
      - api
-      - dev-tls
    volumes:
      - ./certs:/certs:ro
    labels:
      - traefik.http.routers.vulcain.rule=Host(`vulcain.${DOMAIN_NAME}`)
+    <<: *network

  db:
    image: postgres:12-alpine
    environment:
      - POSTGRES_DB=${DB_NAME}
      - POSTGRES_PASSWORD=${DB_PASS}
      - POSTGRES_USER=${DB_USER}
    volumes:
      - db-data:/var/lib/postgresql/data:rw
    labels:
      - traefik.http.routers.db.rule=Host(`db.${DOMAIN_NAME}`)
+    <<: *network

  mercure:
    image: dunglas/mercure
    environment:
#      - ACME_HOSTS=${DOMAIN_NAME}
#      - CERT_FILE=/certs/localhost.crt
#      - KEY_FILE=/certs/localhost.key
      - JWT_KEY=${JWT_KEY}
      - ALLOW_ANONYMOUS=1
      - USE_FORWARDED_HEADERS=true
      - CORS_ALLOWED_ORIGINS=*
      - READ_TIMEOUT=0s
      - WRITE_TIMEOUT=0s
      - PUBLISH_ALLOWED_ORIGINS=*
-    depends_on:
-      - dev-tls
    volumes:
      - ./certs:/certs:ro
    labels:
      - traefik.http.routers.mercure.rule=Host(`mercure.${DOMAIN_NAME}`)
+    <<: *network

  client:
    build:
      context: ./client
      target: api_platform_client_development
      cache_from:
        - ${CLIENT_IMAGE:-quay.io/api-platform/client}
    image: ${CLIENT_IMAGE:-quay.io/api-platform/client}
    tty: true
    environment:
      - API_PLATFORM_CLIENT_GENERATOR_ENTRYPOINT=http://api
      - API_PLATFORM_CLIENT_GENERATOR_OUTPUT=src
+      # You should remove this variable from .env into client folder
+      - REACT_APP_API_ENTRYPOINT=${HTTP_OR_SSL}api.${DOMAIN_NAME}
-    depends_on:
-      - dev-tls
    volumes:
      - ./client:/usr/src/client:rw,cached
    expose:
      - 3000
    labels:
      - traefik.http.routers.client.rule=Host(`client.${DOMAIN_NAME}`)
      - traefik.http.services.client.loadbalancer.server.port=3000
+    <<: *network

  admin:
    build:
      context: ./admin
      target: api_platform_admin_development
      cache_from:
        - ${ADMIN_IMAGE:-quay.io/api-platform/admin}
    image: ${ADMIN_IMAGE:-quay.io/api-platform/admin}
    tty: true
-    depends_on:
-      - dev-tls
+    environment:
+      # You should remove this variable from .env into client folder
+      - REACT_APP_API_ENTRYPOINT=${HTTP_OR_SSL}api.${DOMAIN_NAME}
    volumes:
      - ./admin:/usr/src/admin:rw,cached
    expose:
      - 3000
    labels:
      - traefik.http.routers.admin.rule=Host(`admin.${DOMAIN_NAME}`)
      - traefik.http.services.admin.loadbalancer.server.port=3000
+    <<: *network

volumes:
  db-data: {}

+networks:
+  api_platform_network:
+    external: true
```

Finally, some environment variables must be defined, here is an example of a `.env` file to set them:

```dotenv
CONTAINER_REGISTRY_BASE=quay.io/api-platform
DOMAIN_NAME=localhost
HTTP_OR_SSL=https://
DB_NAME=api-platform-db-name
DB_PASS=YouMustChangeThisPassword
DB_USER=api-platform
JWT_KEY=!UnsecureChangeMe!
SUBDOMAINS_LIST=(admin|api|mercure)
```

This way, you can configure your main variables into one single file.

## Multiple Instances

If you want to run multiple API Platform instances on the same server and behind only one Træfik instance, you'll have to define different service names for each service to avoid named conflicts error since Træfik v2.0.  
To achieve that, by setting only one more environment variable, you'll be able to make each instance unique. Here is a working example below:

```dotenv
# /anywhere/first/api-platform/.env
#...
RANDOM_UNIQUE_KEY=yourUniqueKeyForYourFirstInstance
```

```dotenv
# /anywhere/second/api-platform/.env
#...
RANDOM_UNIQUE_KEY=yourUniqueKeyForYourSecondInstance
```

Then update each traefik http routers names and services following this sample for admin

```yaml
# /anywhere/first/api-plaform/docker-compose.yml
# ...
    labels:
      - traefik.http.routers.admin-${RANDOM_UNIQUE_KEY}.rule=Host(`admin.${DOMAIN_NAME}`)
      - traefik.http.services.admin-${RANDOM_UNIQUE_KEY}.loadbalancer.server.port=3000
```

## More Generic Approach

Here is a fully working sample for Træfik generic config with a little script using docker-compose override approach.  
We assume that you've set `EXPOSE 3000` in your client and admin Dockerfile.

Create a new `init-dc.sh` which contains the generation code that will be written in `docker-compose.override.yaml` file.

```bash
#!/bin/sh
# /anywhere/api-platform/init-dc.sh

services=("admin:admin." "api:api." "mercure:mercure.") # Define your services keys following this format: "{container key}:{sub DNS}". To define root DNS write nothing after the colon
text="version: '3.4'
services:"

for k in "${services[@]}" ; do
    key=${k%%:*}
    value=${k#*:}
    text+="
  $key:
    labels:
      - traefik.http.routers.$key-\${RANDOM_UNIQUE_KEY}.entrypoints=web-secure
      - traefik.http.routers.$key-\${RANDOM_UNIQUE_KEY}.rule=Host(\`${value}\${DOMAIN_NAME}\`)
      - traefik.http.routers.$key-\${RANDOM_UNIQUE_KEY}.tls=true
      - traefik.http.routers.$key-\${RANDOM_UNIQUE_KEY}.tls.domains[0].main=\${DOMAIN_NAME}
      - traefik.http.routers.$key-\${RANDOM_UNIQUE_KEY}.tls.domains[0].sans=*.\${DOMAIN_NAME}
      - traefik.http.routers.$key-\${RANDOM_UNIQUE_KEY}.tls.certresolver=sample
"
done

echo "$text" > ./docker-compose.override.yml
```

Write this minimal configuration into your `traefik.toml` file:

```toml
# /anywhere/traefik/traefik.toml
[providers.docker]
  endpoint = "unix:///var/run/docker.sock"

[api]
  insecure = true
  dashboard = true
  debug = true

[entryPoints.web]
  address = ":80"
  [entryPoints.web.http]
    [entryPoints.web.http.redirections]
      [entryPoints.web.http.redirections.entryPoint]
        to = "web-secure"
        scheme = "https"

[entryPoints.web-secure]
  address = ":443"
```

Then after that update respectively your API Platform and Træfik `docker-compose.yml` following these examples below.

```yaml
# /anywhere/api-platform/docker-compose.yml
version: '3.4'

x-cache:
  &cache
  cache_from:
    - ${CONTAINER_REGISTRY_BASE}/php
    - ${CONTAINER_REGISTRY_BASE}/nginx
    - ${CONTAINER_REGISTRY_BASE}/varnish

x-network:
  &network
  networks:
    - api_platform_network

services:
  php:
    build:
      context: ./api
      target: api_platform_php
      <<: *cache
    image: ${PHP_IMAGE:-quay.io/api-platform/php}
    environment:
      # You should remove these variables from .env into api folder
      - TRUSTED_HOSTS=^(((${SUBDOMAINS_LIST}\.)?${DOMAIN_NAME})|api)$$
      - CORS_ALLOW_ORIGIN=^${HTTP_OR_SSL}(${SUBDOMAINS_LIST}.)?${DOMAIN_NAME}$$
      - DATABASE_URL=postgres://${DB_USER}:${DB_PASS}@db/${DB_NAME}
      - MERCURE_SUBSCRIBE_URL=http://mercure/.well-known/mercure
      - MERCURE_PUBLISH_URL=http://mercure/.well-known/mercure
      - MERCURE_JWT_TOKEN=${JWT_KEY}
    healthcheck:
      interval: 10s
      timeout: 3s
      retries: 3
      start_period: 30s
    depends_on:
      - db
    volumes:
      - ./api:/srv/api:rw,cached
      - ./api/docker/php/conf.d/api-platform.dev.ini:/usr/local/etc/php/conf.d/api-platform.ini
      - ./certs:/certs:ro
    <<: *network

  api:
    build:
      context: ./api
      target: api_platform_nginx
      <<: *cache
    image: ${NGINX_IMAGE:-quay.io/api-platform/nginx}
    depends_on:
      - php
    volumes:
      - ./api/public:/srv/api/public:ro
    <<: *network

  vulcain:
    image: dunglas/vulcain
    environment:
      - CERT_FILE=/certs/localhost.crt
      - KEY_FILE=/certs/localhost.key
      - UPSTREAM=http://api
    depends_on:
      - api
    volumes:
      - ./certs:/certs:ro
    <<: *network

  db:
    image: postgres:12-alpine
    environment:
      - POSTGRES_DB=${DB_NAME}
      - POSTGRES_PASSWORD=${DB_PASS}
      - POSTGRES_USER=${DB_USER}
    volumes:
      - db-data:/var/lib/postgresql/data:rw
    <<: *network

  mercure:
    image: dunglas/mercure
    environment:
#      - ACME_HOSTS=${DOMAIN_NAME}
#      - CERT_FILE=/certs/localhost.crt
#      - KEY_FILE=/certs/localhost.key
      - JWT_KEY=${JWT_KEY}
      - ALLOW_ANONYMOUS=1
      - USE_FORWARDED_HEADERS=true
      - CORS_ALLOWED_ORIGINS=*
      - READ_TIMEOUT=0s
      - WRITE_TIMEOUT=0s
      - PUBLISH_ALLOWED_ORIGINS=*
    volumes:
      - ./certs:/certs:ro
    <<: *network

  client:
    build:
      context: ./client
      target: api_platform_client_development
      cache_from:
        - ${CLIENT_IMAGE:-quay.io/api-platform/client}
    image: ${CLIENT_IMAGE:-quay.io/api-platform/client}
    tty: true
    environment:
      - API_PLATFORM_CLIENT_GENERATOR_ENTRYPOINT=http://api
      - API_PLATFORM_CLIENT_GENERATOR_OUTPUT=src
      # You should remove this variable from .env into client folder
      - REACT_APP_API_ENTRYPOINT=${HTTP_OR_SSL}api.${DOMAIN_NAME}
    volumes:
      - ./client:/usr/src/client:rw,cached
    <<: *network

  admin:
    build:
      context: ./admin
      target: api_platform_admin_development
      cache_from:
        - ${ADMIN_IMAGE:-quay.io/api-platform/admin}
    image: ${ADMIN_IMAGE:-quay.io/api-platform/admin}
    tty: true
    environment:
      # You should remove this variable from .env into client folder
      - REACT_APP_API_ENTRYPOINT=${HTTP_OR_SSL}api.${DOMAIN_NAME}
    volumes:
      - ./admin:/usr/src/admin:rw,cached
    <<: *network

volumes:
  db-data: {}

networks:
  api_platform_network:
    external: true
```

```yaml
# /anywhere/traefik/docker-compose.yml
version: '3.4'

x-network:
  &network
  networks:
    - api_platform_network

services:
  traefik:
    image: traefik:latest
    ports:
      - target: 80
        published: 80
        protocol: tcp
      - target: 443
        published: 443
        protocol: tcp
      - target: 8080
        published: 8080
        protocol: tcp

    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - ./traefik.toml:/etc/traefik/traefik.toml
      - ./acme.json:/acme.json
    <<: *network

networks:
  api_platform_network:
    external: true
```

For a more detailed step-by-step configuration, take a look at [this repository](https://github.com/darkweak/WorkshopContainous) which include Fail2ban link to Træfik instance.

# Implement Traefik Into API Platform Dockerized

> An open-source reverse proxy and load balancer for HTTP and TCP-based applications that is easy, dynamic, automatic, fast, full-featured, production proven, provides metrics, and integrates with every major cluster technology.
>
> —https://traefik.io

## Basic Implementation

This tutorial will help you to define your own routes for your client, api and more generally for your containers.

Use this custom API Platform `docker-compose.yml` file which implements ready-to-use Traefik container configuration. Override
ports and add labels to tell Traefik to listen on the routes mentioned and redirect routes to specified container.

A few points to note:
* `--api` Tells Traefik to generate a browser view to watch containers and IP/DNS associated easier  
* `--docker` Tells Traefik to listen on Docker Api  
* `labels:` Key for Traefik configuration into Docker integration

  ```yaml
  services:
  #  ...
    api:
      labels: 
        - traefik.frontend.rule=Host:api.localhost
  ``` 

  The API DNS will be specified with `traefik.frontend.rule=Host:your.host` (here api.localhost)
* `--traefik.port=3000` The port specified to Traefik will be exposed by the container (here the React app exposes the 3000 port)

```yaml
# docker-compose.yml
version: '3.4'

x-cache:
  &cache
  cache_from:
    - ${CONTAINER_REGISTRY_BASE}/php
    - ${CONTAINER_REGISTRY_BASE}/nginx
    - ${CONTAINER_REGISTRY_BASE}/varnish

services:
  traefik:
    image: traefik
    command: --api --docker
    ports:
      - "80:80" #All HTTP access will be caught by Traefik
      - "443:443" #All HTTPS access will be caught by Traefik
      - "8080:8080" #Access Traefik webview
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock

  php:
    image: ${CONTAINER_REGISTRY_BASE}/php
    build:
      context: ./api
      target: api_platform_php
      <<: *cache
    depends_on:
      - db
    # Comment out these volumes in production
    volumes:
      - ./api:/srv/api:rw,cached
      # If you develop on Linux, uncomment the following line to use a bind-mounted host directory instead
      # - ./api/var:/srv/api/var:rw

  api:
    image: ${CONTAINER_REGISTRY_BASE}/nginx
    labels:
      - traefik.frontend.rule=Host:api.localhost
    build:
      context: ./api
      target: api_platform_nginx
      <<: *cache
    depends_on:
      - php
    # Comment out this volume in production
    volumes:
      - ./api/public:/srv/api/public:ro

  cache-proxy:
    image: ${CONTAINER_REGISTRY_BASE}/varnish
    build:
      context: ./api
      target: api_platform_varnish
      <<: *cache
    depends_on:
      - api
    volumes:
      - ./api/docker/varnish/conf:/usr/local/etc/varnish:ro
    tmpfs:
      - /usr/local/var/varnish:exec
    labels:
      - traefik.frontend.rule=Host:cache.localhost

  db:
    # In production, you may want to use a managed database service
    image: postgres:10-alpine
    labels:
      - traefik.frontend.rule=Host:db.localhost
    environment:
      - POSTGRES_DB=api
      - POSTGRES_USER=api-platform
      # You should definitely change the password in production
      - POSTGRES_PASSWORD=!ChangeMe!
    volumes:
      - db-data:/var/lib/postgresql/data:rw
      # You may use a bind-mounted host directory instead, so that it is harder to accidentally remove the volume and lose all your data!
      # - ./docker/db/data:/var/lib/postgresql/data:rw
    ports:
      - "5432:5432"

  mercure:
    # In production, you may want to use the managed version of Mercure, https://mercure.rocks
    image: dunglas/mercure
    environment:
      # You should definitely change all these values in production
      - JWT_KEY=!UnsecureChangeMe!
      - ALLOW_ANONYMOUS=1
      - CORS_ALLOWED_ORIGINS=*
      - PUBLISH_ALLOWED_ORIGINS=http://mercure.localhost
      - DEMO=1
    labels:
      - traefik.frontend.rule=Host:localhost

  client:
    # Use a static website hosting service in production
    # See https://github.com/facebookincubator/create-react-app/blob/master/packages/react-scripts/template/README.mddeployment
    image: ${CONTAINER_REGISTRY_BASE}/client
    build:
      context: ./client
      cache_from:
        - ${CONTAINER_REGISTRY_BASE}/client
    env_file:
      - ./client/.env
    volumes:
      - ./client:/usr/src/client:rw,cached
      - /usr/src/client/node_modules
    expose:
      - 3000
    labels:
      - traefik.port=3000
      - traefik.frontend.rule=Host:localhost

  admin:
    # Use a static website hosting service in production
    # See https://facebook.github.io/create-react-app/docs/deployment
    image: ${CONTAINER_REGISTRY_BASE}/admin
    build:
      context: ./admin
      cache_from:
        - ${CONTAINER_REGISTRY_BASE}/admin
    volumes:
      - ./admin:/usr/src/admin:rw,cached
      - /usr/src/admin/node_modules
    expose:
      - 3000
    labels:
      - traefik.port=3000
      - traefik.frontend.rule=Host:admin.localhost

volumes:
  db-data: {}
```

Don't forget the db-data, or the database won't work in this dockerized solution.

`localhost` is a reserved domain referred to in your `/etc/hosts`. 
If you want to implement custom DNS such as production DNS in local, just add them at the end of your `/etc/host` file like that: 

```
# /etc/hosts
# ...

127.0.0.1       your.domain.com
```

If you do that, you'll have to update the `CORS_ALLOW_ORIGIN` environment variable `api/.env` to accept the specified URL.

## Known Issues

If your network is of type B, it may conflict with the Traefik sub-network.

## Going Further

As this Traefik configuration listens on 80 and 443 ports, you can run only 1 Traefik instance per server. However, you may want to run multiple API Platform projects on same server. To deal with it, you'll have to externalize the Traefik configuration to another `docker-compose.yml` file, anywhere on your server.

Here is a working example:

```yaml
# /somewhere/docker-compose.yml
version: '3.4'

services:
  traefik:
    image: traefik
    command: --api --docker
    ports:
      - "80:80"
      - "443:443"
      - "8080:8080"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
# load a TOML configuration file and to generate Let's Encrypt certificated as explained in https://docs.traefik.io/user-guide/docker-and-lets-encrypt/
#      - ./traefik.toml:/traefik.toml
#      - ./acme.json:/acme.json
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
# docker-compose.yml
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
# Uncomment these lines only if you want to run one api-platform instance using traefik
-  traefik:
-    image: traefik:latest
-    command: --api --docker
-    ports:
-      - "80:80"
-      - "443:443"
-    volumes:
-      - /var/run/docker.sock:/var/run/docker.sock
-    <<: *network

  php:
    image: ${CONTAINER_REGISTRY_BASE}/php
    build:
      context: ./api
      target: api_platform_php
      <<: *cache
    depends_on: 
      - db
+    environment:
+      # You should remove these variables from .env into api folder
+      - TRUSTED_HOSTS=^(((${SUBDOMAINS_LIST}\.)?${DOMAIN_NAME})|api)$$
+      - CORS_ALLOW_ORIGIN=^${HTTP_OR_SSL}(${SUBDOMAINS_LIST}.)?${DOMAIN_NAME}$$
+      - DATABASE_URL=postgres://${DB_USER}:${DB_PASS}@db/${DB_NAME}
+      - MERCURE_SUBSCRIBE_URL=${HTTP_OR_SSL}mercure.${DOMAIN_NAME}$$
+      - MERCURE_PUBLISH_URL=${HTTP_OR_SSL}mercure.${DOMAIN_NAME}$$
+      - MERCURE_JWT_SECRET=${JWT_KEY}
    volumes:
      - ./api:/srv/api:rw,cached
+    <<: *network

  api:
    image: ${CONTAINER_REGISTRY_BASE}/nginx
    build:
      context: ./api
      target: api_platform_nginx
      <<: *cache
    depends_on:
      - php
    volumes:
      - ./api/public:/srv/api/public:ro
    labels:
      - traefik.frontend.rule=Host:api.${DOMAIN_NAME}
+    <<: *network

  cache-proxy:
    image: ${CONTAINER_REGISTRY_BASE}/varnish
    build:
      context: ./api
      target: api_platform_varnish
      <<: *cache
    depends_on:
      - api
    volumes:
      - ./api/docker/varnish/conf:/usr/local/etc/varnish:ro
    tmpfs:
      - /usr/local/var/varnish:exec
    labels:
      - traefik.frontend.rule=Host:cache.${DOMAIN_NAME}
+    <<: *network

  db:
    image: postgres:10-alpine
    environment:
      - POSTGRES_DB=${DB_NAME}
      - POSTGRES_USER=${DB_USER}
      - POSTGRES_PASSWORD=${DB_PASS}
    volumes:
      - db-data:/var/lib/postgresql/data:rw
+    <<: *network

  mercure:
    image: dunglas/mercure
    environment:
      - JWT_KEY=${JWT_KEY}
      - ALLOW_ANONYMOUS=0
      - CORS_ALLOWED_ORIGINS=^${HTTP_OR_SSL}(${SUBDOMAINS_LIST}.)?${DOMAIN_NAME}$$
      - PUBLISH_ALLOWED_ORIGINS=${HTTP_OR_SSL}
      - DEMO=1
    labels:
      - traefik.frontend.rule=Host:mercure.${DOMAIN_NAME}
+    <<: *network

  client:
    image: ${CONTAINER_REGISTRY_BASE}/client
    build:
      context: ./client
      cache_from:
        - ${CONTAINER_REGISTRY_BASE}/client
    volumes:
      - ./client:/usr/src/client:rw,cached
      - /usr/src/client/node_modules
    expose:
      - 3000
    labels:
      - traefik.frontend.rule=Host:${DOMAIN_NAME},www.${DOMAIN_NAME}
      - traefik.port=3000
    environment:
      # You should remove this variable from .env into client folder
      - REACT_APP_API_ENTRYPOINT=${HTTP_OR_SSL}api.${DOMAIN_NAME}
+    <<: *network

  admin:
    image: ${CONTAINER_REGISTRY_BASE}/admin
    build:
      context: ./admin
      cache_from:
        - ${CONTAINER_REGISTRY_BASE}/admin
    environment:
      # You should remove this variable from .env into admin folder
      - REACT_APP_API_ENTRYPOINT=${HTTP_OR_SSL}api.${DOMAIN_NAME}
    volumes:
      - ./admin:/usr/src/admin:rw,cached
      - /usr/src/admin/node_modules
    expose:
      - 3000
    labels:
      - traefik.frontend.rule=Host:admin.${DOMAIN_NAME}
      - traefik.port=3000
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
HTTP_OR_SSL=http://
DB_NAME=api-platform-db-name
DB_PASS=YouMustChangeThisPassword
DB_USER=api-platform
JWT_KEY=!UnsecureChangeMe!
SUBDOMAINS_LIST=(admin|api|cache|mercure|www)
```

This way, you can configure your main variables into one single file.

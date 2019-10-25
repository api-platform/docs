# Implement Traefik Into API Platform Dockerized

> An open-source reverse proxy and load balancer for HTTP and TCP-based applications that is easy, dynamic, automatic, fast, full-featured, production proven, provides metrics, and integrates with every major cluster technology.
>
> —https://traefik.io

## Basic Implementation

This tutorial will help you to define your own routes for your client, api and more generally for your containers.

Use this custom API Platform `docker-compose.yml` file which implements ready-to-use Traefik container configuration. Override
ports and add labels to tell Traefik to listen on the routes mentioned and redirect routes to specified container.

```yaml
# docker-compose.yml
version: '3.7'

x-cache:
  &cache
  cache_from:
    - ${CONTAINER_REGISTRY_BASE}/php
    - ${CONTAINER_REGISTRY_BASE}/nginx
    - ${CONTAINER_REGISTRY_BASE}/varnish

services:
  traefik:
    image: traefik:2.0.2
    command:
      - --api.insecure # dashboard URL: http://localhost:8080/dashboard/
      - --entrypoints.web.address=:80 # http port
      - --entrypoints.websecure.address=:443 # https port
      - --entrypoints.postgresql.address=:5432 # postgresql port
      - --log.level=INFO # log level
      - --providers.docker # enable docker provider
      - --providers.docker.exposedbydefault=false # disable automatic provisioning
    ports:
      - "80:80"
      - "8080:8080"
      - "443:443"
      - "5432:5432"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro

  php:
    image: ${CONTAINER_REGISTRY_BASE}/php
    build:
      context: ./api
      target: api_platform_php
      <<: *cache
    depends_on:
      - db
    environment:
      - TRUSTED_HOSTS=^(((${SUBDOMAINS_LIST}\.)?${DOMAIN_NAME})|api)$$
      - CORS_ALLOW_ORIGIN=^${HTTP_OR_SSL}(${SUBDOMAINS_LIST}.)?${DOMAIN_NAME}$$
      - DATABASE_URL=postgres://${DB_USER}:${DB_PASS}@db/${DB_NAME}
      - MERCURE_SUBSCRIBE_URL=${HTTP_OR_SSL}mercure.${DOMAIN_NAME}$$
      - MERCURE_PUBLISH_URL=${HTTP_OR_SSL}mercure.${DOMAIN_NAME}$$
      - MERCURE_JWT_SECRET=${JWT_KEY}
    # Comment out these volumes in production
    volumes:
      - ./api:/srv/api:rw,cached
      # If you develop on Linux, uncomment the following line to use a bind-mounted host directory instead
      # - ./api/var:/srv/api/var:rw

  api:
    image: ${CONTAINER_REGISTRY_BASE}/nginx
    labels:
      - traefik.enable=true
      - traefik.http.routers.api.rule=Host(`api.localhost`)
      - traefik.http.routers.api.entrypoints=websecure
      - traefik.http.routers.api.tls
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
      - traefik.enable=true
      - traefik.http.routers.cache-proxy.rule=Host(`cache.localhost`)
      - traefik.http.routers.cache-proxy.entrypoints=websecure
      - traefik.http.routers.cache-proxy.tls

  db:
    # In production, you may want to use a managed database service
    image: postgres:11.5-alpine
    labels:
      - traefik.enable=true
      - traefik.tcp.routers.db.rule=HostSNI(`*`)
      - traefik.tcp.routers.db.entrypoints=postgresql
      - traefik.tcp.services.db.loadbalancer.server.port=5432
    environment:
      - POSTGRES_DB=api
      - POSTGRES_USER=api-platform
      # You should definitely change the password in production
      - POSTGRES_PASSWORD=!ChangeMe!
    volumes:
      - db-data:/var/lib/postgresql/data:rw
      # You may use a bind-mounted host directory instead, so that it is harder to accidentally remove the volume and lose all your data!
      # - ./docker/db/data:/var/lib/postgresql/data:rw

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
      - traefik.enable=true
      - traefik.http.routers.mercure.rule=Host(`mercure.localhost`)
      - traefik.http.routers.mercure.entrypoints=websecure
      - traefik.http.routers.mercure.tls

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
      - traefik.enable=true
      - traefik.http.routers.client.rule=Host(`localhost`)
      - traefik.http.routers.client.entrypoints=websecure
      - traefik.http.routers.client.tls
      - traefik.http.services.client.loadbalancer.server.port=3000

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
      - traefik.enable=true
      - traefik.http.routers.admin.rule=Host(`admin.localhost`)
      - traefik.http.routers.admin.entrypoints=websecure
      - traefik.http.routers.admin.tls
      - traefik.http.services.admin.loadbalancer.server.port=3000

volumes:
  db-data: {}
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
This way, you can configure your main variables into one single file. Don't forget the db-data, or the database won't work in this dockerized solution.

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

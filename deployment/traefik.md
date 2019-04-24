# Implement Traefik Into API Platform Dockerized

## Basic Implementation

[Traefik](https://traefik.io) is a reverse proxy / load balancer that's easy, dynamic, automatic, fast, full-featured, open source, production proven, and that provides metrics and integrates with every major cluster technology.

This tool will help you to define your own routes for your client, api and more generally for your containers.

Use this custom API Platform `docker-compose.yml` file which implements ready-to-use Traefik container configuration.  
Override ports and add labels to tell Traefik to listen on the routes mentionned and redirect routes to specified container.


`--api` Tells Traefik to generate a browser view to watch containers and IP/DNS associated easier  
`--docker` Tells Traefik to listen on Docker Api  
`--docker.domain=localhost` The main DNS will be on localhost  
`labels:` Key for Traefik configuration into Docker integration  
```yaml
services:
#  ...
  api:
    labels: 
      - "traefik.frontend.rule=Host:api.localhost"
``` 
The API DNS will be specified with `traefik.frontend.rule=Host:your.host` (here api.localhost)  

`--traefik.port=3000` The port specified to Traefik will be exposed by the container (here the React app exposes the 3000 port)  


```yaml
version: '3.4'

x-cache:
  &cache
  cache_from:
    - ${CONTAINER_REGISTRY_BASE}/php
    - ${CONTAINER_REGISTRY_BASE}/nginx
    - ${CONTAINER_REGISTRY_BASE}/varnish

services:
  reverse-proxy:
    image: traefik
    command: --api --docker --docker.domain=localhost
    ports:
      - "80:80" #All HTTP access will be caught by Traefik
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
      - "traefik.frontend.rule=Host:api.localhost"
    build:
      context: ./api
      target: api_platform_nginx
      <<: *cache
    depends_on:
      - php
    # Comment out this volume in production
    volumes:
      - ./api/public:/srv/api/public:ro
      
  db:
    # In production, you may want to use a managed database service
    image: postgres:10-alpine
    labels:
      - "traefik.frontend.rule=Host:db.localhost"
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

  client:
    # Use a static website hosting service in production
    # See https://facebook.github.io/create-react-app/docs/deployment
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
    labels:
      - "traefik.port=3000"
      - "traefik.frontend.rule=Host:localhost"
      
  mercure:
    # In production, you may want to use the managed version of Mercure, https://mercure.rocks
    image: dunglas/mercure
    environment:
      # You should definitely change all these values in production
      - JWT_KEY=!UnsecureChangeMe!
      - ALLOW_ANONYMOUS=1
      - CORS_ALLOWED_ORIGINS=*
      - PUBLISH_ALLOWED_ORIGINS=http://localhost:1337,https://localhost:1338
      - DEMO=1
    labels:
      - "traefik.frontend.rule=Host:mercure.localhost"
      
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
    labels:
      - "traefik.frontend.rule=Host:admin.localhost"
      
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

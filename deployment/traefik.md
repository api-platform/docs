# Implement Traefik Into API Platform Dockerized

## Basic Implementation

[Traefik](https://traefik.io) is a reverse proxy / load balancer that's easy, dynamic, automatic, fast, full-featured, open source, production proven, providing metrics, and integrating with every major cluster technology.

This tool will help you to define your own routes for your client, api and more generally for your containers.

Use this custom API Platform `docker-compose.yml` file which implements ready-to-use Traefik container configuration.  
Override ports and add labels to tell Traefik to listen the routes mentionned and redirect routes to specified container.


`--api` Tell Traefik to generate a browser view to watch containers and IP/DNS associated easier  
`--docker` Tell Traefik to listen docker api  
`--docker.domain=localhost` The main DNS will be on localhost  
`labels:` Key for Traefik configuration into Docker integration  
```yaml
services:
#  ...
  api:
    labels: 
      - "traefik.frontend.rule=Host:api.localhost"
``` 
The api DNS will be specified with `traefik.frontend.rule=Host:your.host` (here api.localhost)  

`--traefik.port=3000` Port specified to Traefik will be exopsed by container (here React app expose the 3000 port)  


```yaml
version: '3.4'

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
    depends_on:
      - db
    env_file:
      - ./api/.env
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
    depends_on:
      - php
    # Comment out this volume in production
    volumes:
      - ./api/public:/srv/api/public:ro

  db:
    # In production, you may want to use a managed database service
    image: postgres:9.6-alpine
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
    # See https://github.com/facebookincubator/create-react-app/blob/master/packages/react-scripts/template/README.mddeployment
    image: ${CONTAINER_REGISTRY_BASE}/client
    build:
      context: ./client
    env_file:
      - ./client/.env
    volumes:
      - ./client:/usr/src/client:rw,cached
      - /usr/src/client/node_modules
    expose:
      - 3000
    labels:
      - "traefik.port=3000"
      - "traefik.frontend.rule=Host:localhost"

volumes:
  db-data: {}
```

Don't forget the db-data, then database won't work in this dockerized solution.

`localhost` is a reserved domain referred in your `/etc/hosts`. 
If you want to implement custom DNS such as production DNS in local, just put them at the end of your `/etc/host` file like that: 

```
# /etc/hosts
# ...

127.0.0.1       your.domain.com
```

If you do that, you'll have to update the `CORS_ALLOW_ORIGIN` environment variable `api/.env` to accept the specified URL.

## Known Issues

If your network is of type B, it may conflict with the traefik sub-network.

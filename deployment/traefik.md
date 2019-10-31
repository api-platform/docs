# Implement Træfik Into API Platform Dockerized

> An open-source reverse proxy and load balancer for HTTP and TCP-based applications that is easy, dynamic, automatic, fast, full-featured, production proven, provides metrics, and integrates with every major cluster technology.
>
> —https://traefik.io

## Basic Implementation

This tutorial will help you to define your own routes for your client, api and more generally for your containers.

Use this custom API Platform `docker-compose.yml` file which implements ready-to-use Træfik container configuration. Override
ports and add labels to tell Træfik to listen on the routes mentioned and redirect routes to specified container.

A few points to note:
* `--api.insecure` Tells Træfik to generate a browser view to watch containers and IP/DNS associated easier  
* `--providers.docker` Tells Træfik to listen on Docker Api  
* `labels:` Key for Træfik configuration into Docker integration

  ```yaml
  services:
  #  ...
    api:
      labels: 
        - traefik.http.routers.your-service.rule=Host:api.localhost
  ``` 

  The API DNS will be specified with `traefik.http.routers.your-service.rule=Host("your.host")` (here api.localhost)
* `--traefik.http.services.your-service.loadBalancer.port=3000` The port specified to Træfik will be exposed by the container (here the React app exposes the 3000 port). But if your container expose only one port it's useless to define it.

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
    image: traefik:v2.0
    command: --api.insecure --providers.docker
    ports:
      - "80:80" #All HTTP access will be caught by Træfik
      - "443:443" #All HTTPS access will be caught by Træfik
      - "8080:8080" #Access Træfik webview
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
    build:
      context: ./api
      target: api_platform_nginx
      <<: *cache
    depends_on:
      - php
    # Comment out this volume in production
    volumes:
      - ./api/public:/srv/api/public:ro
    labels:
      - traefik.http.routers.api.rule=Host(`api.localhost`)

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
      - traefik.http.routers.cache.rule=Host(`cache.localhost`)

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
      - traefik.http.routers.mercure.rule=Host(`mercure.localhost`)

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
      - traefik.http.routers.client.rule=Host(`localhost`, `www.localhost`)
      - traefik.http.services.client.loadBalancer.port=3000

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
      - traefik.http.routers.admin.rule=Host(`admin.localhost`)
      - traefik.http.services.admin.loadBalancer.port=3000

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

If your network is of type B, it may conflict with the Træfik sub-network.

## Going Further

As this Tæefik configuration listens on 80 and 443 ports, you can run only 1 Træfik instance per server. However, you may want to run multiple API Platform projects on same server. To deal with it, you'll have to externalize the Træfik configuration to another `docker-compose.yml` file, anywhere on your server.

Here is a working example:

```yaml
# /somewhere/docker-compose.yml
version: '3.4'

services:
  traefik:
    image: traefik:v2.0
    command: --api.insecure --providers.docker
    ports:
      - "80:80"
      - "443:443"
      - "8080:8080"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
# load a TOML configuration file and to generate Let's Encrypt certificated as explained in https://docs.traefik.io/user-guide/docker-and-lets-encrypt/
# If you uncomment this line, you won't be able to use command key anymore since Træfik V2.0 allow only one of this
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

+x-react-env:
+  - &react-env
+    environment:
+      - REACT_APP_API_ENTRYPOINT=${HTTP_OR_SSL}api.${DOMAIN_NAME}

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
      - traefik.http.routers.api.rule=Host(`api.${DOMAIN_NAME}`)
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
      - traefik.http.routers.cache.rule=Host(`cache.${DOMAIN_NAME}`)
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
      - traefik.http.routers.mercure.rule=Host(`mercure.${DOMAIN_NAME}`)
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
      - traefik.http.routers.client.rule=HostRegexp(`{subdomains:(www.)?}${DOMAIN_NAME}`)
      - traefik.http.services.client.loadBalancer.port=3000
+    <<: *network
+    <<: *react-env

  admin:
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
      - traefik.http.routers.admin.rule=Host(`admin.${DOMAIN_NAME}`)
      - traefik.http.services.admin.loadBalancer.port=3000
+    <<: *network
+    <<: *react-env

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

### Multiple API Platform instances in separated folders

Since Træfik V2.0, the router is not dynamic anymore. You have to define one unique key for your service. To be able to work with 2 API Platform instances you have to use two different folders.

For exemple, we will work with one instance in `first` directory and the second in `second` directory. We just have to update labels `routers` and `services` service names to have unique keys.

```yaml
# /first/docker-compose.yml

# ...

x-network:
  &network
  networks:
    - api_platform_network_1

services:
  # ... php configuration

  api:
    # ... api configuration
    labels:
      - traefik.http.routers.api-first.rule=Host(`api.${DOMAIN_NAME}`)
    <<: *network

  cache-proxy:
    # ... cache configuration
    labels:
      - traefik.http.routers.cache-first.rule=Host(`cache.${DOMAIN_NAME}`)
    <<: *network

  # ... db configuration

  mercure:
    # ... mercure configuration
    labels:
      - traefik.http.routers.mercure-first.rule=Host(`mercure.${DOMAIN_NAME}`)
    <<: *network

  client:
    # ... client configuration
    labels:
      - traefik.http.routers.client-first.rule=HostRegexp(`{subdomains:(www.)?}${DOMAIN_NAME}`)
      - traefik.http.services.client-first.loadBalancer.port=3000
    <<: *network

  admin:
    # ... client configuration
    labels:
      - traefik.http.routers.admin-first.rule=HostRegexp(`{subdomains:(www.)?}${DOMAIN_NAME}`)
      - traefik.http.services.admin-first.loadBalancer.port=3000
    <<: *network

# ... 

networks:
  api_platform_network_1:
    external: true
```

```yaml
# /second/docker-compose.yml

# ...

x-network:
  &network
  networks:
    - api_platform_network_2

services:
  # ... php configuration

  api:
    # ... api configuration
    labels:
      - traefik.http.routers.api-second.rule=Host(`api.${DOMAIN_NAME}`)
    <<: *network

  cache-proxy:
    # ... cache configuration
    labels:
      - traefik.http.routers.cache-second.rule=Host(`cache.${DOMAIN_NAME}`)
    <<: *network

  # ... db configuration

  mercure:
    # ... mercure configuration
    labels:
      - traefik.http.routers.mercure-second.rule=Host(`mercure.${DOMAIN_NAME}`)
    <<: *network

  client:
    # ... client configuration
    labels:
      - traefik.http.routers.client-second.rule=HostRegexp(`{subdomains:(www.)?}${DOMAIN_NAME}`)
      - traefik.http.services.client-second.loadBalancer.port=3000
    <<: *network

  admin:
    # ... client configuration
    labels:
      - traefik.http.routers.admin-second.rule=HostRegexp(`{subdomains:(www.)?}${DOMAIN_NAME}`)
      - traefik.http.services.admin-second.loadBalancer.port=3000
    <<: *network

# ...

networks:
  api_platform_network_2:
    external: true
```

Of course, don't forget to add networks to Træfik instance, if not, you won't be able to access to your services.

```yaml
# /somewhere/docker-compose.yml

# ...

networks:
  # ...
  api_platform_network_1:
    external: true
  api_platform_network_2:
    external: true
```


Then update `first` and `second` `.env` files respectively

```dotenv
# /first/.env
DOMAIN_NAME=myfirstdomain.com
# ... unchanged other variables
```

```dotenv
# /second/.env
DOMAIN_NAME=myseconddomain.com
# ... Other variables are unchanged
```

Then as you can see, the template for service naming is `traefik.http.[routers|services].servicename-[directory-name].[any]`.  
So, we can define a variable naming the folder or unique key like that:

```yaml
# /instance/docker-compose.yml

# ...

services:
  # ... php configuration

  api:
    # ... api configuration
    labels:
      - traefik.http.routers.api-${INSTANCE_KEY}.rule=Host(`api.${DOMAIN_NAME}`)

  cache-proxy:
    # ... cache configuration
    labels:
      - traefik.http.routers.cache-${INSTANCE_KEY}.rule=Host(`cache.${DOMAIN_NAME}`)

  # ... db configuration

  mercure:
    # ... mercure configuration
    labels:
      - traefik.http.routers.mercure-${INSTANCE_KEY}.rule=Host(`mercure.${DOMAIN_NAME}`)

  client:
    # ... client configuration
    labels:
      - traefik.http.routers.client-${INSTANCE_KEY}.rule=HostRegexp(`{subdomains:(www.)?}${DOMAIN_NAME}`)
      - traefik.http.services.client-${INSTANCE_KEY}.loadBalancer.port=3000

  admin:
    # ... client configuration
    labels:
      - traefik.http.routers.admin-${INSTANCE_KEY}.rule=HostRegexp(`{subdomains:(www.)?}${DOMAIN_NAME}`)
      - traefik.http.services.admin-${INSTANCE_KEY}.loadBalancer.port=3000

# ...
```

```dotenv
# /instance/.env
INSTANCE_KEY=somewhere
# ... Other variables are unchanged
```

You can now handle multiple API Platform instances configuring only 8 environment variables.

## Handle Træfik logs and link to Fail2ban

To avoid bruteforce, we will implement a Fail2ban instance linked to Træfik and API Platform.  
If you want to try API Platform with multiple instances and fail2ban protection, follow [this link](https://github.com/darkweak/WorkshopContainous).

First, we have to declare a Fail2ban instance. We will link it to the externalized Træfik instance.
This Fail2ban instance is an alpine based container. We need to define the `network_mode` to `host` because it had to be able to edit iptables then it won't be linked to networks because it has to be external.
In log folder you'll find different logs from api or at least Træfik error logs

```yaml
# /somewhere/docker-compose.yml
version: '3.4'
services:
  traefik:
    image: traefik:v2.0
    ports:
      - "80:80"
      - "8080:8080"
      - "443:443"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - ./traefik.toml:/etc/traefik/traefik.toml
      - ./log:/var/log
    networks:
      - api_platform_network
  
  fail2ban:
    image: alpine:latest
    build:
      context: .
      dockerfile: ./Dockerfile-fail2ban
    env_file:
      - ./fail2ban.env
    network_mode: "host"
    cap_add:
      - NET_ADMIN
      - NET_RAW
    volumes:
      - ./log:/var/log:ro
      - ./data:/data

networks:
  api_platform_network:
    external: true
  # ...
```

After that, define a `traefik.toml` file to configure this instance

```toml
# /somewhere/traefik.toml
# Tell Træfik to listen docker, same than commands
[providers.docker]
  endpoint = "unix:///var/run/docker.sock"

# Enable Træfik API, dashboard, allow access to Web UI and run in debug mode
[api]
  insecure = true
  dashboard = true
  debug = true

[entryPoints]
  [entryPoints.web]
    address = ":80"
  [entryPoints.web-secure]
    address = ":443"

[accessLog]
  filePath = "/var/log/traefik/access.log"
  [accessLog.filters]
    # Tell Træfik to handle all 404 domain not found and range from 400 to 403. Change it to your need!
    statusCodes = ["404", "400-403"]
```

From the previous given configuration, Træfik is able to write logs handling 400 to 404 response error codes.  
Then the logs are written in `/somewhere/log` path directory, then you can link your fail2ban on them as we shared log folder in fail2ban volume service declaration.

For the full configuration refer to [this github project](https://github.com/darkweak/WorkshopContainous).

#   Deploying with Docker Compose

While [Docker Compose](https://docs.docker.com/compose/) is mainly known and used in a development environment, it [can
actually be used in production too](https://docs.docker.com/compose/production/). This is especially suitable for prototyping
or small-scale deployments, where the robustness (and the associated complexity) of [Kubernetes](kubernetes.md) is not
required.

It is recommended that you build the Docker images in a [CI (continuous integration)](https://en.wikipedia.org/wiki/Continuous_integration)
job, or failing which, on your development machine. The built images should then be pushed to a [container registry](https://docs.docker.com/registry/introduction/),
e.g. [Docker Hub](https://hub.docker.com/), [Google Container Registry](https://cloud.google.com/container-registry/),
[GitLab Container Registry](https://docs.gitlab.com/ee/user/packages/container_registry/). On the production server, you
would pull the pre-built images from the container registry. This maintains a separation of concerns between the build
environment and the production environment.

##  Installing the Docker Compose Setup for Production

If you are using the [API Platform Distribution](../distribution/index.md), we provide a [ready-to-deploy Docker Compose
setup for production](https://github.com/api-platform/docker-compose-prod), with (optional) [Let's Encrypt](https://letsencrypt.org/)
integration.

It is designed to be used as a companion to the distribution, and as such it needs to be placed inside a subdirectory at
the top level of the distribution project:

    $ wget -O - https://github.com/api-platform/docker-compose-prod/archive/master.tar.gz | tar -xzf - && mv docker-compose-prod-master docker-compose-prod
    $ git add docker-compose-prod

Then commit the changes to your git repository. The `docker-compose-prod` directory should be checked in.

##  Building and Pushing the Docker Images

These steps should be performed in a CI job (recommended) or on your development machine.

1. Make sure the environment variables required for the build are set.

    If you are building the images in a CI job, these environment variables should be set as part of your CI job's environment.

    If you are building on your development machine, you could set the environment variables in the `.env` file at the
    top level of the distribution project (not to be confused with `api/.env` which is used by the Symfony application).
    For example:

    ```
    ADMIN_IMAGE=registry.example.com/api-platform/admin
    CLIENT_IMAGE=registry.example.com/api-platform/client
    NGINX_IMAGE=registry.example.com/api-platform/nginx
    PHP_IMAGE=registry.example.com/api-platform/php
    REACT_APP_API_ENTRYPOINT=https://api.example.com
    VARNISH_IMAGE=registry.example.com/api-platform/varnish
    ```

    **Note**: `REACT_APP_API_ENTRYPOINT` must be an exact match of the target domain name where your API will be accessed
    from, since its value is [embedded during build time](https://create-react-app.dev/docs/adding-custom-environment-variables).
    See [this discussion for possible workarounds](https://github.com/facebook/create-react-app/issues/2353) if this limitation
    is unacceptable for your project.

2. Build the Docker images:

        $ docker-compose -f docker-compose-prod/docker-compose.build.yml pull --ignore-pull-failures
        $ docker-compose -f docker-compose-prod/docker-compose.build.yml build --pull

3. Push the built images to the container registry:

        $ docker-compose -f docker-compose-prod/docker-compose.build.yml push

##  Pulling the Docker Images and Running the Services

These steps should be performed on the production server.

1. Make sure the environment variables required are set.

    You could set the environment variables in the `.env` file at the top level of the distribution project (not to be
    confused with `api/.env` which is used by the Symfony application). For example:

    ```
    ADMIN_HOST=admin.example.com
    ADMIN_IMAGE=registry.example.com/api-platform/admin
    API_HOST=api.example.com
    APP_SECRET=3c857494cfcc42c700dfb7a6
    CLIENT_HOST=example.com,www.example.com
    CLIENT_IMAGE=registry.example.com/api-platform/client
    CORS_ALLOW_ORIGIN=^https://(?:\w+\.)?example\.com$
    DATABASE_URL=postgres://api-platform:4e3bc2766fe81df300d56481@db/api
    MERCURE_ALLOW_ANONYMOUS=0
    MERCURE_CORS_ALLOWED_ORIGINS=https://example.com,https://admin.example.com
    MERCURE_HOST=mercure.example.com
    MERCURE_JWT_KEY=4121344212538417de3e2118
    MERCURE_JWT_SECRET=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJtZXJjdXJlIjp7InN1YnNjcmliZSI6WyJmb28iLCJiYXIiXSwicHVibGlzaCI6WyJmb28iXX19.B0MuTRMPLrut4Nt3wxVvLtfWB_y189VEpWMlSmIQABQ
    MERCURE_SUBSCRIBE_URL=https://mercure.example.com/hub
    NGINX_IMAGE=registry.example.com/api-platform/nginx
    PHP_IMAGE=registry.example.com/api-platform/php
    POSTGRES_PASSWORD=4e3bc2766fe81df300d56481
    REACT_APP_API_ENTRYPOINT=https://api.example.com
    TRUSTED_HOSTS=^(?:localhost|api|api\.example\.com)$
    VARNISH_IMAGE=registry.example.com/api-platform/varnish
    ```

    **Important**: Please make sure to change all the passwords, keys and secret values to your own.

2. Set up a redirect from e.g. `www.example.com` to `example.com`:

        $ mkdir -p docker-compose-prod/docker/nginx-proxy/vhost.d
        $ echo 'return 301 https://example.com$request_uri;' > docker-compose-prod/docker/nginx-proxy/vhost.d/www.example.com

    **Note**: If you do not want such a redirect, or you want it to be the other way round, please adapt to suit your needs.

3. *(optional)* Set up the Let's Encrypt integration.

    **Note**: If you are using Cloudflare, you might consider using their [free SSL/TLS encryption](https://www.cloudflare.com/ssl/)
    setup as a simpler alternative. But if you would prefer to have full control, read on.

    Make sure the environment variables required for the Let's Encrypt integration are set.

    You could set the environment variables in the `.env` file at the top level of the distribution project (not to be
    confused with `api/.env` which is used by the Symfony application). For example:

    ```
    LETSENCRYPT_USER_MAIL=user@example.com
    LEXICON_CLOUDFLARE_AUTH_TOKEN=9e06358f74cbce70602c22fc3279f0aee3077
    LEXICON_CLOUDFLARE_AUTH_USERNAME=user@example.com
    ```

    **Note**: If you are not using [Cloudflare DNS](https://www.cloudflare.com/dns/), please see [the documentation on
    how to pass the correct environment variables to Lexicon](https://github.com/adferrand/docker-letsencrypt-dns#configuring-dns-provider-and-authentication-to-dns-api).

    Configure the (sub)domains for which you want certificate(s) to be issued for in `docker-compose-prod/docker/letsencrypt/domains.conf`.
    For example, to request a wildcard certificate for `*.example.com` and `example.com`:

    ```
    *.example.com example.com autorestart-containers=api-platform_nginx-proxy_1
    ```

    **Note**: Replace the `api-platform` prefix in `api-platform_nginx-proxy_1` with your [Docker Compose project name
    (it defaults to the project directory name)](https://docs.docker.com/compose/reference/envvars/#compose_project_name).

4. Pull the Docker images.

    If you are **not** using the (optional) Let's Encrypt integration:

        $ docker-compose -f docker-compose-prod/docker-compose.yml pull

    If you are using the (optional) Let's Encrypt integration:

        $ docker-compose -f docker-compose-prod/docker-compose.yml -f docker-compose-prod/docker-compose.letsencrypt.yml pull

5. Bring up the services.

    If you are **not** using the (optional) Let's Encrypt integration:

        $ docker-compose -f docker-compose-prod/docker-compose.yml up -d

    If you are using the (optional) Let's Encrypt integration:

        $ docker-compose -f docker-compose-prod/docker-compose.yml -f docker-compose-prod/docker-compose.letsencrypt.yml up -d

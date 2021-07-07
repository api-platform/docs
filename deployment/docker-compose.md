# Deploying with Docker Compose

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

## Deploying for Production

To build images for production, you have to use the `docker-compose.prod.yml` file with Docker Compose.

## Building and Pushing the Docker Images

These steps should be performed in a CI job (recommended) or on your development machine.

1. Make sure the environment variables required for the build are set.

    If you are building the images in a CI job, these environment variables should be set as part of your CI job's environment.

    If you are building on your development machine, you could set the environment variables in the `.env` file at the
    top level of the distribution project (not to be confused with `api/.env` which is used by the Symfony application).
    For example:

    ```shell
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

    ```shell
    docker-compose -f docker-compose.yml -f docker-compose.prod.yml build --pull
    ```

3. Push the built images to the container registry:

    ```shell
    docker-compose -f docker-compose.yml -f docker-compose.prod.yml push
    ```

## Pulling the Docker Images and Running the Services

These steps should be performed on the production server.

1. Make sure the environment variables required are set.

    You could set the environment variables in the `.env` file at the top level of the distribution project (not to be
    confused with `api/.env` which is used by the Symfony application). For example:

    ```shell
    SERVER_NAME=api.example.com
    MERCURE_PUBLISHER_JWT_KEY=someKey
    MERCURE_SUBSCRIBER_JWT_KEY=someKey
    ```

    **Important**: Please make sure to change all the passwords, keys and secret values to your own.

2. Pull the Docker images.

    ```console
    docker-compose -f docker-compose.yml -f docker-compose.prod.yml pull
    ```

3. Bring up the services.

    ```console
    docker-compose -f docker-compose.yml -f docker-compose.prod.yml up -d
    ```

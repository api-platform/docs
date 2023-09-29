# Deploying to a Kubernetes Cluster

[Kubernetes](https://kubernetes.io/) has become the most popular way to deploy, run and manage containers in production.
Both [Google Cloud Platform](https://cloud.google.com/kubernetes-engine/), [Microsoft Azure](https://azure.microsoft.com/en-us/services/container-service/kubernetes/)
and [Amazon Web Services](https://aws.amazon.com/eks/) provide managed Kubernetes environment.

[The official API Platform distribution](../distribution/index.md) contains a built-in [Helm](https://helm.sh/) (the k8s
package manager) chart to deploy in a wink on any of these platforms.

## Preparing Your Cluster and Your Local Machine

1. Create a Kubernetes cluster on your preferred Cloud provider or install Kubernetes locally on your servers
2. Install [Helm](https://helm.sh/) locally and on your cluster following their documentation
3. Be sure to be connected to the right Kubernetes container
4. Update the Helm repo: `helm repo update`

## Creating and Publishing the Docker Images

1. Build the PHP and Nginx Docker images:

```
        docker build -t gcr.io/test-api-platform/php -t gcr.io/test-api-platform/php:latest api
        docker build -t gcr.io/test-api-platform/nginx -t gcr.io/test-api-platform/nginx:latest -f api/Dockerfile.nginx api
        docker build -t gcr.io/test-api-platform/varnish -t gcr.io/test-api-platform/varnish:latest -f api/Dockerfile.varnish api
```

2. Push your images to your Docker registry, example with [Google Container Registry](https://cloud.google.com/container-registry/):

```
        gcloud docker -- push gcr.io/test-api-platform/php
        gcloud docker -- push gcr.io/test-api-platform/nginx
        gcloud docker -- push gcr.io/test-api-platform/varnish
```

## Deploying

Firstly you need to update helm dependencies by running:

```
    helm dependency update ./api/helm/api
```

You are now ready to deploy the API!

Deploy your API to the container:

```
    helm install ./api/helm/api --namespace=baz --name baz \
        --set php.repository=gcr.io/test-api-platform/php \
        --set nginx.repository=gcr.io/test-api-platform/nginx \
        --set secret=MyAppSecretKey \
        --set postgresql.postgresPassword=MyPgPassword \
        --set postgresql.persistence.enabled=true \
        --set corsAllowOrigin='^https?://[a-z\]*\.mywebsite.com$'
```

If you prefer to use a managed DBMS like [Heroku Postgres](https://www.heroku.com/postgres) or
[Google Cloud SQL](https://cloud.google.com/sql/docs/postgres/) (recommended):

```
    helm install --name api ./api/helm/api \
        # ...
        --set postgresql.enabled=false \
        --set postgresql.url=pgsql://username:password@host/database?serverVersion=9.6
```

If you want to use a managed Varnish such as [Fastly](https://www.fastly.com) for the invalidation cache mechanism
provided by API Platform:

```
    helm install --name api ./api/helm/api \
        # ...
        --set varnish.enabled=false \
        --set varnish.url=https://myvarnish.com
```

Finally, build the `client` and `admin` JavaScript apps and [deploy them on a static
website hosting service](https://github.com/facebookincubator/create-react-app/blob/master/packages/react-scripts/template/README.md#deployment).

## Initializing the Database

Before running your application for the first time, be sure to create the database schema:

```
    PHP_POD=$(kubectl --namespace=bar get pods -l app=php -o jsonpath="{.items[0].metadata.name}")
    kubectl --namespace=bar exec -it $PHP_POD -- bin/console doctrine:schema:create
```
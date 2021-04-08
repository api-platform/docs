# Deploying to a Kubernetes Cluster

[Kubernetes](https://kubernetes.io/) has become the most popular way to deploy, run and manage containers in production.
[Google Cloud Platform](https://cloud.google.com/kubernetes-engine/), [Microsoft Azure](https://azure.microsoft.com/en-us/services/container-service/kubernetes/)
and [Amazon Web Services](https://aws.amazon.com/eks/) and many more companies provide managed Kubernetes environment.

[The official API Platform distribution](../distribution/index.md) contains a built-in [Helm](https://helm.sh/) (the k8s
package manager) chart to deploy in a wink on any of these platforms.

This guide is based on Helm 3 and the caddy/php/pwa/postgres Stack introduced with API Platform 2.6

## Preparing Your Cluster and Your Local Machine

1. Create a Kubernetes cluster on your preferred Cloud provider or install Kubernetes locally on your servers by example with [kubeadm](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/)
2. Install [Helm](https://helm.sh/) (preferred version >= 3) `locally` and on your `cluster` following their [documentation](https://helm.sh/docs/intro/install/)
3. Be sure to be connected to the right Kubernetes container
   `kubectl config view` [Details](https://kubernetes.io/docs/concepts/configuration/organize-cluster-access-kubeconfig/)
   e.g. for Google Cloud running: `gcloud config get-value core/project`
4. Update the Helm repo on your local machine: `helm repo update`
   If you have a fresh installation of helm, you do not have any repositories yet.

Working-Dir: Your local installation of api-platform. Default /api-platform/

## Creating and Publishing the Docker Images

### Example with the [Google Container Registry](https://cloud.google.com/container-registry/)

#### 1. Build the PHP and Caddy Docker images and tag them

    docker build -t gcr.io/test-api-platform/php -t gcr.io/test-api-platform/php:latest api --target api_platform_php
    docker build -t gcr.io/test-api-platform/caddy -t gcr.io/test-api-platform/caddy:latest api --target api_platform_caddy
    docker build -t gcr.io/test-api-platform/pwa -t gcr.io/test-api-platform/pwa:latest pwa --target api_platform_pwa_prod

#### 2. Push your images to your Docker registry

    gcloud auth configure-docker
    docker push gcr.io/test-api-platform/php
    docker push gcr.io/test-api-platform/caddy
    docker push gcr.io/test-api-platform/pwa

## Deploying with Helm 3

### 1. Check the Helm version

    helm version

If you are using version 2.x follow this [guide to migrate Helm to v3](https://helm.sh/docs/topics/v2_v3_migration/#helm)

### 2. Firstly you need to update helm dependencies by running

    helm dependency update ./helm/api-platform

### 3. Optional: If you made changes to the Helm chart, check if its format is correct

    helm lint ./helm/api-platform

### 4. Deploy your API to the container

    helm install api-platform ./helm/api-platform --namespace=default \
        --set "php.image.repository=gcr.io/test-api-platform/php" \
        --set php.image.tag=latest \
        --set "caddy.image.repository=gcr.io/test-api-platform/caddy" \
        --set caddy.image.tag=latest \
        --set "pwa.image.repository=gcr.io/test-api-platform/pwa" \
        --set pwa.image.tag=latest \
        --set php.appSecret='!ChangeMe!' \
        --set postgresql.postgresqlPassword='!ChangeMe!' \
        --set postgresql.persistence.enabled=true \
        --set corsAllowOrigin='^https?://[a-z\]*\.mywebsite.com$'

You can add the parameter `--dry-run` to check upfront if anything is correct.
Replace the values with the image parameters from the stage above.
The parameter `php.appSecret` is the `AppSecret` from ./.env
Fill the rest of the values with the correct settings.
For available options see /helm/api-platform/values.yaml.

If you prefer to use a managed DBMS like [Heroku Postgres](https://www.heroku.com/postgres) or
[Google Cloud SQL](https://cloud.google.com/sql/docs/postgres/) (recommended):

    helm install api-platform ./helm/api-platform \
        # ...
        --set postgresql.enabled=false \
        --set postgresql.url=pgsql://username:password@host/database?serverVersion=13

Finally, build the `pwa` (client and admin) JavaScript apps and [deploy them on a static
website hosting service](https://create-react-app.dev/docs/deployment/).

## Initializing the Database

Before running your application for the first time, be sure to create the database schema:

    CADDY_PHP_POD=$(kubectl --namespace=default get pods -l app.kubernetes.io/name=api-platform -o jsonpath="{.items[0].metadata.name}")
    kubectl --namespace=default exec -it $CADDY_PHP_POD -c api-platform-php -- bin/console doctrine:schema:create

## Helm 2: Tiller RBAC Issue

Tiller is the server component of Helm, which is removed in Helm 3.
[Migrating Helm v2 to v3](https://helm.sh/docs/topics/v2_v3_migration/#helm)

We noticed that some tiller RBAC trouble occurred. You can usually resolve it by running:

    kubectl create serviceaccount --namespace kube-system tiller
      serviceaccount "tiller" created

    kubectl create clusterrolebinding tiller-cluster-rule --clusterrole=cluster-admin --serviceaccount=kube-system:tiller
      clusterrolebinding "tiller-cluster-rule" created

    kubectl patch deploy --namespace kube-system tiller-deploy -p '{"spec":{"template":{"spec":{"serviceAccount":"tiller"}}}}'
      deployment "tiller-deploy" patched

Please, see the [related issue](https://github.com/kubernetes/helm/issues/3130) for further details / information.
You can also take a look at the [related documentation](https://github.com/kubernetes/helm/blob/master/docs/rbac.md)

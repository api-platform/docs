# Deploying to a Kubernetes Cluster

[Kubernetes](https://kubernetes.io/) has become the most popular way to deploy, run and manage containers in production.
[Google Cloud Platform](https://cloud.google.com/kubernetes-engine/), [Microsoft Azure](https://azure.microsoft.com/en-us/services/container-service/kubernetes/)
and [Amazon Web Services](https://aws.amazon.com/eks/) and many more companies provide managed Kubernetes environment.

[The official API Platform distribution](../distribution/index.md) contains a built-in [Helm](https://helm.sh/) (the k8s
package manager) chart to deploy in a wink on any of these platforms.

This guide is based on Helm 3.

## Preparing Your Cluster and Your Local Machine

1. Create a Kubernetes cluster on your preferred Cloud provider or install Kubernetes locally on your server, for example with [kubeadm](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/)
2. Install [Helm 3](https://helm.sh/) `locally` and on your `cluster` following their [documentation](https://helm.sh/docs/intro/install/)
3. Be sure to be connected to the right Kubernetes container
   `kubectl config view` [Details](https://kubernetes.io/docs/concepts/configuration/organize-cluster-access-kubeconfig/)
   e.g. for Google Cloud running: `gcloud config get-value core/project`

Working-Dir: Your local installation of api-platform. Default /api-platform/

## Creating and Publishing the Docker Images

### Example with the [Google Container Registry](https://cloud.google.com/container-registry/) and [Google Cloud Platform](https://cloud.google.com/kubernetes-engine/)

Change the name "test-api-platform" to your Google project ID (not the project name).
[Quickstart Google Cloud](https://cloud.google.com/sdk/docs/quickstart?hl=de)
If you do not have gcloud yet, install it with these command.

    curl https://sdk.cloud.google.com | bash

#### 1. Build the PHP and Caddy Docker images and tag them

Versioning: The 0.1.0 is the version. These value should be same as the attribut "appVersion" in Chart.yaml
Infos for [Google Container pulling and pushing](https://cloud.google.com/container-registry/docs/pushing-and-pulling)

    docker build -t gcr.io/test-api-platform/php:0.1.0 -t gcr.io/test-api-platform/php:latest api --target api_platform_php
    docker build -t gcr.io/test-api-platform/caddy:0.1.0 -t gcr.io/test-api-platform/caddy:latest api --target api_platform_caddy
    docker build -t gcr.io/test-api-platform/pwa:0.1.0 -t gcr.io/test-api-platform/pwa:latest pwa --target api_platform_pwa_prod

#### 2. Push your images to your Docker registry

    gcloud auth configure-docker
    docker push gcr.io/test-api-platform/php
    docker push gcr.io/test-api-platform/caddy
    docker push gcr.io/test-api-platform/pwa

Optional push the version images

    docker push gcr.io/test-api-platform/php:0.1.0 
    docker push gcr.io/test-api-platform/caddy:0.1.0 
    docker push gcr.io/test-api-platform/pwa:0.1.0 

The result should look similar to these images.

![Example of Google Images - Overview](images/google-image-overview.png)
![Example of Google Caddy Image - Details](images/google-image-caddy-details.png)

## Deploying with Helm 3

### 1. Check the Helm version

    helm version

If you are using version 2.x follow this [guide to migrate Helm to v3](https://helm.sh/docs/topics/v2_v3_migration/#helm)

### 2. Firstly you need to update helm dependencies by running

    helm dependency update ./helm/api-platform

This will create a folder /charts/ and add all dependencies there.
Actual this is [bitnami/postgresql](https://bitnami.com/stack/postgresql/helm), a file postgresql-[VERSION].tgz is created.

### 3. Optional: If you made changes to the Helm chart, check if its format is correct

    helm lint ./helm/api-platform

### 4. Deploy your API to the container

    helm upgrade api-platform ./helm/api-platform --namespace=default \
        --install \
        --set "php.image.repository=gcr.io/test-api-platform/php" \
        --set php.image.tag=latest \
        --set "caddy.image.repository=gcr.io/test-api-platform/caddy" \
        --set caddy.image.tag=latest \
        --set "pwa.image.repository=gcr.io/test-api-platform/pwa" \
        --set pwa.image.tag=latest \
        --set php.appSecret='!ChangeMe!' \
        --set postgresql.postgresqlPassword='!ChangeMe!' \
        --set postgresql.persistence.enabled=true \
        --set "corsAllowOrigin=^https?://[a-z\]*\.mywebsite.com$"

The " are necessary for Windows. Use ^ on Windows instead of \ to split commands into multiple lines.
You can add the parameter `--dry-run` to check upfront if anything is correct.
Replace the values with the image parameters from the stage above.
The parameter `php.appSecret` is the `AppSecret` from ./.env
Fill the rest of the values with the correct settings.
For available options see /helm/api-platform/values.yaml.
If you want a test deploy you can set corsAllowOrigin='*'

After a successful installation, there is a message at the end.
You can copy these commands and execute them to set a port-forwarding and
get access on your local machine to the deploy. See image below.

![Deploy Result](images/deploy-result.png)

If you prefer to use a managed DBMS like [Heroku Postgres](https://www.heroku.com/postgres) or
[Google Cloud SQL](https://cloud.google.com/sql/docs/postgres/) (recommended):

    helm upgrade api-platform ./helm/api-platform \
        # ...
        --set postgresql.enabled=false \
        --set postgresql.url=pgsql://username:password@host/database?serverVersion=13

Finally, build the `pwa` (client and admin) JavaScript apps and [deploy them on a static
website hosting service](https://create-react-app.dev/docs/deployment/).

## Access the container

You can access the php container of the pod with the following command.
In this example the symfony console is called.

    CADDY_PHP_POD=$(kubectl --namespace=default get pods -l app.kubernetes.io/name=api-platform -o jsonpath="{.items[0].metadata.name}")
    kubectl --namespace=default exec -it $CADDY_PHP_POD -c api-platform-php -- bin/console

## Caution for system architecture

If the pods do not run, and you get the following error from google kubernetes engine logs,
there is probably a problem with the system architecture.
`standard_init_linux.go:211: exec user process caused "exec format error`
Build the images with the same system architecture as the cluster runs.
Example: Building with Mac M1 with arm64 leads to problems. Most cluster will run with x86_64.
Solution: <https://blog.jaimyn.dev/how-to-build-multi-architecture-docker-images-on-an-m1-mac/>

## Updates

There are 2 main upgrade strategies.

### 1. Always version your images (recommended)

Change the version in the attribut "appVersion" in Chart.yaml and tag the images with this version.
You can upgrade with the same command from the installation and pass all parameters.

### 2. Use :latest tags

Infos about [best practices for tagging images for kubernetes](https://kubernetes.io/docs/concepts/containers/images/)
You have to use the *.image.pullPolicy=Always see the last 3 parameters.

    helm upgrade api-platform ./helm/api-platform --namespace=default \
    --set "php.image.repository=gcr.io/test-api-platform/php" \
    --set php.image.tag=latest \
    --set "caddy.image.repository=gcr.io/test-api-platform/caddy" \
    --set caddy.image.tag=latest \
    --set "pwa.image.repository=gcr.io/test-api-platform/pwa" \
    --set pwa.image.tag=latest \
    --set php.appSecret='!ChangeMe!' \
    --set postgresql.postgresqlPassword='!ChangeMe!' \
    --set postgresql.persistence.enabled=true \
    --set "corsAllowOrigin=^https?://[a-z\]*\.mywebsite.com$" \
    --set php.image.pullPolicy=Always \
    --set caddy.image.pullPolicy=Always \
    --set pwa.image.pullPolicy=Always

## GitHub Actions Example for deployment

You can find a [complete deploy command for GKE](https://github.com/api-platform/demo/blob/main/.github/workflows/deploy.yml) on the [demo project](https://github.com/api-platform/demo/):

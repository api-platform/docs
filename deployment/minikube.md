# Deploying to Minikube

## Install Minikube

If you have no existing installation of Minikube on your computer, [follow the official tutorial](https://minikube.sigs.k8s.io/docs/start/).

When Minikube is installed, start the cluster:

```console
minikube start --addons registry --addons dashboard
```

The previous command starts Minikube with a Docker registry (we'll use it in the next step) and with the Kubernetes dashboard.

If you use Mac or Windows, [refer to the documentation](https://minikube.sigs.k8s.io/docs/handbook/registry/#docker-on-macos) to learn how to expose the Docker registry installed as an addon on the port 5000 of the host.

Finally, [install Helm](https://helm.sh/docs/intro/install/). We'll use it to deploy the application in the cluster thanks to the chart provided in the API Platform distribution.

## Building and Pushing Docker Images

First, build the images:

```console
docker build -t localhost:5000/php api --target api_platform_php
docker build -t localhost:5000/caddy api --target api_platform_caddy
docker build -t localhost:5000/pwa pwa --target api_platform_pwa_prod
```

Then push the images in the registry installed in Minikube:

```console
docker push localhost:5000/php
docker push localhost:5000/caddy
docker push localhost:5000/pwa
```

## Deploying

Finally, deploy the project using the Helm chart:

```console
$ helm install my-project helm/api-platform \
  --set php.image.repository=localhost:5000/php \
  --set php.image.tag=latest \
  --set caddy.image.repository=localhost:5000/caddy \
  --set caddy.image.tag=latest \
  --set pwa.image.repository=localhost:5000/pwa \
  --set pwa.image.tag=latest
```

Copy and paste the commands displayed in the terminal to enable the port forwarding then go to `http://localhost:8080` to access your application!

Run `minikube dashboard` at any moment to see the state of your deployments.

## Using Skaffold

Skaffold is a tool for Kubernetes development: [https://skaffold.dev/](https://skaffold.dev/).

It will build and deploy automatically your app in Kubernetes and apply every changes. The default configuration use minikube and helm. More configurations are available in Skaffold documentation.

First, install the [skaffold CLI](https://skaffold.dev/docs/install/#standalone-binary).

Then, run minikube:

```bash
minikube start
```

Add Skaffold configuration in the file `./helm/skaffold.yaml`. You can find a [complete configuration file for minikube](https://github.com/api-platform/api-platform/blob/main/helm/skaffold.yaml) with its [Helm values override](https://github.com/api-platform/api-platform/blob/main/helm/skaffold-values.yaml).

Finally, go to the helm folder, and run skaffold in dev mode:

```bash
cd ./helm
skaffold dev
```

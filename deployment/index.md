# Deploying API Platform Applications

API Platform apps are super easy to deploy in production thanks to the [Docker Compose definetion](docker-compose.md) and to the [Kubernetes chart](kubernetes.md) we provide.

We strongly recommend using Kubernetes or Docker Compose to deploy your apps.

If you want to play with a local Kubernetes cluster, read [how to deploy an API Platform project on Minikube](minikube.md).

If you don't want to use Docker, keep in mind that the server application of API Platform is a standard Symfony project,
while the Progressive Web Application is a standard Next.js project:

<p align="center" class="symfonycasts"><a href="https://symfonycasts.com/screencast/ansistrano?cid=apip"><img src="../distribution/images/symfonycasts-player.png" alt="JWT screencast"><br>Watch the Animated Deployment with Ansistrano screencast</a></p>

* [Deploying the Symfony application](https://symfony.com/doc/current/deployment.html)
* [Deployin the Next.js application](https://nextjs.org/docs/deployment)

Alternatively, you may want to deploy API Platform on a PaaS (Platform as a Service):

* [Deploying the server application of API Platform on Heroku](heroku.md)
* [Deploying API Platform on Platform.sh (outdated)](https://platform.sh/blog/deploy-api-platform-on-platformsh)

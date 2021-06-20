# Deploying API Platform Applications

API Platform apps are super easy to deploy in production on most cloud providers thanks to the native integration with
[Kubernetes](kubernetes.md).

The server part of API Platform is basically a standard Symfony application, which you can also easily [deploy on your
own servers](https://symfony.com/doc/current/deployment.html). Documentation for deploying on various container [orchestration](https://en.wikipedia.org/wiki/Orchestration_(computing))
tools and PaaS (Platform as a Service) are also available:

* [Deploying to a Kubernetes Cluster](kubernetes.md)
* [Deploying with Docker Compose](docker-compose.md)
* [Deploying on Heroku](heroku.md)
* [Deploying on Platform.sh](https://platform.sh/blog/deploy-api-platform-on-platformsh)

The clients are [Create React App](https://create-react-app.dev/) skeletons. You can deploy them in a wink
on any static website hosting service (including [Netlify](https://www.netlify.com/), [Firebase Hosting](https://firebase.google.com/docs/hosting/),
[GitHub Pages](https://pages.github.com/), or [Amazon S3](https://docs.aws.amazon.com/en_us/AmazonS3/latest/dev/WebsiteHosting.html)
by following [the relevant documentation](https://create-react-app.dev/docs/deployment/).

<p align="center" class="symfonycasts"><a href="https://symfonycasts.com/screencast/ansistrano?cid=apip"><img src="../distribution/images/symfonycasts-player.png" alt="JWT screencast"><br>Watch the Animated Deployment with Ansistrano screencast</a></p>

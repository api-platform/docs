# Deploying API Platform Applications

API Platform apps are super-easy to deploy in production in most cloud providers thanks to the native integration with [Kubernetes](kubernetes.md).

The server part of API Platform is basically a standard Symfony application, [that you can also easily deploy on your own
servers](http://symfony.com/doc/current/deployment.html). 
Documentation entries to deploy on various PaaS (Platform as a Service) are also available:

* [Deploying to a Kubernetes Cluster](kubernetes.md)
* [Deploying on Heroku](heroku.md)
* [Deploying on Platform.sh](https://platform.sh/blog/deploy-api-platform-on-platformsh)

The clients are [Create React App](https://github.com/facebook/create-react-app/) skeletons. You can deploy them in a wink
on any static website hosting service (including [Netlify](https://www.netlify.com/), [Firebase Hosting](https://firebase.google.com/docs/hosting/),
[GitHub Pages](https://pages.github.com/), or [Amazon S3](https://docs.aws.amazon.com/en_us/AmazonS3/latest/dev/WebsiteHosting.html)
by following [the relevant documentation](https://github.com/facebook/create-react-app/blob/master/packages/react-scripts/template/README.md#deployment).

# Enabling Mercure support

API Platform can automatically send real time updates to the currently connected clients (webapps, mobile apps...) using [the Mercure protocol](https://mercure.rocks).

> *Mercure* is a protocol allowing to push data updates to web browsers and other HTTP clients in a convenient, fast, reliable and battery-efficient way. It is especially useful to publish real-time updates of resources served through web APIs, to reactive web and mobile apps.
>
> —https://mercure.rocks

API Platform detects changes made to your Doctrine entities, and sends the updated resources to the Mercure hub.
Then, the Mercure hub dispatches the updates to all connected clients using [Server-sent Events (SSE)](https://developer.mozilla.org/en-US/docs/Web/API/Server-sent_events).

![Mercure subscriptions](images/mercure-subscriptions.png)

## Installation

Mercure support is already installed, configured and enabled in [the API Platform distribution](../../distribution/index.md).
If you use the _distribution_, you have nothing more to do, and you can skip to the next section.

If you have installed API Platform using another method (such as `composer require api`), you need to install a Mercure hub, and the [Symfony MercureBundle](https://github.com/symfony/mercure-bundle):

First, [download and run a Mercure hub](https://github.com/dunglas/mercure#hub-implementation).
Then, install the Symfony bundle:

     $ composer require mercure

Finally, 3 environment variables [must be set](https://symfony.com/doc/current/configuration/external_parameters.html):

* `MERCURE_PUBLISH_URL`: the URL that must be used by API Platform to publish updates to your Mercure hub (can be an internal or a public URL)
* `MERCURE_SUBSCRIBE_URL`: the **public** URL of the Mercure hub that clients will use to subscribe to updates
* `MERCURE_JWT_SECRET`: a valid Mercure [JSON Web Token (JWT)](https://jwt.io/) allowing API Platform to publish updates to the hub

The JWT **must** contain a `mercure.publish` property containing an array of targets.
This array can be empty to allow publishing anonymous updates only.
[Example publisher JWT](https://jwt.io/#debugger-io?token=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJtZXJjdXJlIjp7InN1YnNjcmliZSI6WyJmb28iLCJiYXIiXSwicHVibGlzaCI6WyJmb28iXX19.LRLvirgONK13JgacQ_VbcjySbVhkSmHy3IznH3tA9PM) (demo key: `!UnsecureChangeMe!`).

[Learn more about Mercure authorization.](https://github.com/dunglas/mercure/blob/master/spec/mercure.md#authorization)

# Pushing Live Updates Using the Mercure Protocol

API Platform can automatically send real time updates to the currently connected clients (webapps, mobile apps...) using [the Mercure protocol](https://mercure.rocks).

> *Mercure* is a protocol allowing to push data updates to web browsers and other HTTP clients in a convenient, fast, reliable and battery-efficient way. It is especially useful to publish real-time updates of resources served through web APIs, to reactive web and mobile apps.
>
> —https://mercure.rocks

API Platform detects changes made to your Doctrine entities, and sends the updated resources to the Mercure hub.
Then, the Mercure hub dispatches the updates to all connected clients using [Server-sent Events (SSE)](https://developer.mozilla.org/en-US/docs/Web/API/Server-sent_events).

![Mercure subscriptions](images/mercure-subscriptions.png)

## Installing Mercure Support

Mercure support is already installed, configured and enabled in [the API Platform distribution](../distribution/index.md).
If you use the distribution, you have nothing more to do, and you can skip to the next section.

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

## Pushing the API Updates

Use the `mercure` attribute to hint API Platform that it must dispatch the updates regarding the given resources to the Mercure hub:

```php
<?php
// api/src/Entity/Book.php

namespace App\Entity;

use ApiPlatform\Core\Annotation\ApiResource;

/**
 * @ApiResource(mercure=true)
 */
class Book
{
    // ...
}
```

Then, every time an object of this type is created, updated or deleted, the new version is sent to all connected clients through the Mercure hub.
If the resource has been deleted, only the (now deleted) IRI of the resource is sent to the clients.

In addition, API Platform automatically adds a `Link` HTTP header to all responses related to this resource class.
This header allows smart clients to automatically discover the Mercure hub.

![Mercure subscriptions](images/mercure-discovery.png)

Clients generated using [the API Platform Client Generator](../client-generator/index.md) will use this capability to automatically subscribe to Mercure updates when available:

![Screencast](../client-generator/images/client-generator-demo.gif)

[Learn how to use the discovery capabilities of Mercure in your own clients](https://github.com/dunglas/mercure#examples).

## Dispatching Private Updates (Authorized Mode)

Mercure allows to dispatch [private updates, that will be received only by authorized clients](https://github.com/dunglas/mercure/blob/master/spec/mercure.md#authorization).
To receive this kind of updates, the client must hold a JWT containing at least one *target* marking the update.

Then, hint API Platform to dispatch the updates only to the selected targets:

```php
<?php
// api/src/Entity/Book.php

namespace App\Entity;

use ApiPlatform\Core\Annotation\ApiResource;

/**
 * @ApiResource(mercure={"a-group", "another-group"})
 */
class Book
{
    // ...
}
```

It's also possible to execute an *expression* (using the [Symfony Expression Language component](https://symfony.com/doc/current/components/expression_language.html)), to generate a dynamic list of targets:

```php
<?php
// api/src/Entity/Book.php

namespace App\Entity;

use ApiPlatform\Core\Annotation\ApiResource;

/**
 * @ApiResource(mercure="object.owners")
 */
class Book
{
    public $owners = [];

   // ...
}
```

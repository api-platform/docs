# Creating Async APIs using the Mercure Protocol

API Platform can automatically push the modified version of the resources exposed by the API to the currently connected clients (webapps, mobile apps...) using [the Mercure protocol](https://mercure.rocks).

> *Mercure* is a protocol allowing to push data updates to web browsers and other HTTP clients in a convenient, fast, reliable and battery-efficient way. It is especially useful to publish real-time updates of resources served through web APIs, to reactive web and mobile apps.
>
> —<https://mercure.rocks>

API Platform detects changes made to your Doctrine entities, and sends the updated resources to the Mercure hub.
Then, the Mercure hub dispatches the updates to all connected clients using [Server-sent Events (SSE)](https://developer.mozilla.org/en-US/docs/Web/API/Server-sent_events).

![Mercure subscriptions](images/mercure-subscriptions.png)

## Installing Mercure Support

Mercure support is already installed, configured and enabled in [the API Platform distribution](../distribution/index.md).
If you use the distribution, you have nothing more to do, and you can skip to the next section.

If you have installed API Platform using another method (such as `composer require api`), you need to install a Mercure hub, and the [Symfony MercureBundle](https://symfony.com/doc/current/mercure.html):

First, [download and run a Mercure hub](https://mercure.rocks/docs/hub/install).
Then, install the Symfony bundle:

```console
composer require symfony/mercure-bundle
```

Finally, 3 environment variables [must be set](https://symfony.com/doc/current/configuration/external_parameters.html):

* `MERCURE_PUBLISH_URL`: the URL that must be used by API Platform to publish updates to your Mercure hub (can be an internal or a public URL)
* `MERCURE_SUBSCRIBE_URL`: the **public** URL of the Mercure hub that clients will use to subscribe to updates
* `MERCURE_JWT_TOKEN`: a valid Mercure [JSON Web Token (JWT)](https://jwt.io/) allowing API Platform to publish updates to the hub

The JWT **must** contain a `mercure.publish` property containing an array of topic selectors.
This array can be empty to allow publishing anonymous updates only. It can also be `["*"]` to allow publishing on every topics.
[Example publisher JWT](https://jwt.io/#debugger-io?token=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJtZXJjdXJlIjp7InB1Ymxpc2giOlsiKiJdfX0.obDjwCgqtPuIvwBlTxUEmibbBf0zypKCNzNKP7Op2UM) (demo key: `!ChangeMe!`).

[Learn more about Mercure authorization.](https://mercure.rocks/spec#authorization)

## Pushing the API Updates

Use the `mercure` attribute to hint API Platform that it must dispatch the updates regarding the given resources to the Mercure hub:

```php
<?php
// api/src/Entity/Book.php

namespace App\Entity;

use ApiPlatform\Core\Annotation\ApiResource;

#[ApiResource(mercure: true)]
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

[Learn how to use the discovery capabilities of Mercure in your own clients](https://mercure.rocks/docs/ecosystem/awesome).

## Dispatching Private Updates (Authorized Mode)

Mercure allows to dispatch [private updates, that will be received only by authorized clients](https://mercure.rocks/spec#authorization).
To receive this kind of updates, the client must hold a JWT containing at least one *target selector* matched by the update.

Then, use options to mark the published updates as privates:

```php
<?php
// api/src/Entity/Book.php

namespace App\Entity;

use ApiPlatform\Core\Annotation\ApiResource;

#[ApiResource(mercure: ["private" => true])]
class Book
{
    // ...
}
```

It's also possible to execute an *expression* (using the [Symfony Expression Language component](https://symfony.com/doc/current/components/expression_language.html)), to generate the options dynamically:

```php
<?php
// api/src/Entity/Book.php

namespace App\Entity;

use ApiPlatform\Core\Annotation\ApiResource;

#[ApiResource(mercure: "object.mercureOptions")]
class Book
{
    public $mercureOptions = ['private' => true];

   // ...
}
```

## Available Options

In addition to `private`, the following options are available:

* `topics`: the list of topics of this update, if not the resource IRI is used
* `data`: the content of this update, if not set the content will be the serialization of the resource using the default format
* `id`: the SSE id of this event, if not set the ID will be generated by the mercure Hub
* `type`: the SSE type of this event, if not set this field is omitted
* `retry`: the `retry` field of the SSE, if not set this field is omitted
* `normalization_context`: the specific normalization context to use for the update.

## Dispatching Restrictive Updates (Security Mode)

Use `iri` (iriConverter) and `escape` (rawurlencode) functions to add an alternative topic, in order to restrict a subscriber with `topic_selector` to receive only publications that are authorized (partner match).

> Let's say that a subscriber wants to receive updates concerning all book resources it has access to. The subscriber can use the topic selector <https://example.com/books/{id}> as value of the topic query parameter.
> Adding this same URI template to the mercure.subscribe claim of the JWS presented by the subscriber to the hub would allow this subscriber to receive all updates for all book resources. It is not what we want here: this subscriber is only authorized to access some of these resources.
>
> To solve this problem, the mercure.subscribe claim could contain a topic selector such as: <https://example.com/users/foo/{?topic}>.
>
> The publisher could then take advantage of the previously described behavior by publishing a private update having <https://example.com/books/1> as canonical topic and <https://example.com/users/foo/?topic=https%3A%2F%2Fexample.com%2Fbooks%2F1> as alternate topic.
>
> —<https://mercure.rocks/spec#subscribers>

Below is an example using the `topics` option:

```php
<?php
// api/src/Entity/Book.php

namespace App\Entity;

use ApiPlatform\Core\Annotation\ApiResource;
use ApiPlatform\Core\Api\UrlGeneratorInterface;
use App\Entity\User;

#[ApiResource(
    mercure: [
        'private' => true,
        // the '@=' prefix is required when using expressions for arguments in topics
        'topics' => [
            '@=iri(object)',
            '@=iri(object.getOwner()) ~ "/?topic=" ~ escape(iri(object))',
            '@=iri(object, '.UrlGeneratorInterface::ABS_PATH.')', // you can also change the reference type
            'https://example.com/books/1',
        ],
    ],
)]
class Book
{
    private ?User $owner;

    public function getOwner(): ?User
    {
        return $this->owner;
    }
}
```

Using an *expression* function:

```php
<?php
// api/src/Entity/Book.php

namespace App\Entity;

use ApiPlatform\Core\Annotation\ApiResource;
use App\Entity\User;

#[ApiResource(
    mercure: 'object.getMercureOptions()',
)]
class Book
{
    private ?User $owner;

    public function getMercureOptions(): array
    {
        // the '@=' prefix is required when using expressions for arguments in topics
        $topic1 = '@=iri(object)';
        $topic2 = '@=iri(object.getOwner()) ~ "/?topic=" ~ escape(iri(object))';
        $topic3 = '@=iri(object, '.UrlGeneratorInterface::ABS_PATH.')'; // you can also change the reference type
        $topic4 = 'https://example.com/books/1';

        return [
            'private' => true,
            'topics' => [$topic1, $topic2, $topic3, $topic4],
        ];
    }

    public function getOwner(): ?User
    {
        return $this->owner;
    }
}
```

In this case, the JWT Token for the subscriber should contain:

```json
{
  "mercure": {
    "subscribe": ["https://example.com/users/foo/{?topic}"]
  }
}
```

The subscribe topic should be:
`https://example.com/books/{id}`

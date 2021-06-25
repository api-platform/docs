# The Event System

Note: using Kernel event with API Platform should be mostly limited to tweaking the generated HTTP response. Also, GraphQL is **not supported**.
[For most use cases, better extension points, working both with REST and GraphQL, are available](extending.md).

API Platform Core implements the [Action-Domain-Responder](https://github.com/pmjones/adr) pattern. This implementation
is covered in depth in the [Creating custom operations and controllers](operations.md#creating-custom-operations-and-controllers)
chapter.

Basically, API Platform Core executes an action class that will return an entity or a collection of entities. Then a series
of event listeners are executed which validate the data, persist it in database, serialize it (typically in a JSON-LD document)
and create an HTTP response that will be sent to the client.

To do so, API Platform Core leverages [events triggered by the Symfony HTTP Kernel](https://symfony.com/doc/current/reference/events.html#kernel-events).
You can also hook your own code to those events. There are handy and powerful extension points available at all points
of the request lifecycle.

If you are using Doctrine, lifecycle events ([ORM](https://www.doctrine-project.org/projects/doctrine-orm/en/current/reference/events.html#lifecycle-events), [MongoDB ODM](https://www.doctrine-project.org/projects/doctrine-mongodb-odm/en/latest/reference/events.html#lifecycle-events))
are also available if you want to hook into the persistence layer's object lifecycle.

## Built-in Event Listeners

These built-in event listeners are registered for routes managed by API Platform:

Name                          | Event              | [Pre & Post hooks](#custom-event-listeners) | Priority | Description
------------------------------|--------------------|---------------------------------------------|----------|-------------
`AddFormatListener`           | `kernel.request`   | None                                        | 7        | Guesses the best response format ([content negotiation](content-negotiation.md))
`ReadListener`                | `kernel.request`   | `PRE_READ`, `POST_READ`                     | 4        | Retrieves data from the persistence system using the [data providers](data-providers.md) (`GET`, `PUT`, `PATCH`, `DELETE`)
`DeserializeListener`         | `kernel.request`   | `PRE_DESERIALIZE`, `POST_DESERIALIZE`       | 2        | Deserializes data into a PHP entity (`GET`, `POST`, `DELETE`); updates the entity retrieved using the data provider (`PUT`, `PATCH`)
`DenyAccessListener`          | `kernel.request`   | None                                        | 1        | Enforces [access control](security.md) using Security expressions
`ValidateListener`            | `kernel.view`      | `PRE_VALIDATE`, `POST_VALIDATE`             | 64       | [Validates data](validation.md) (`POST`, `PUT`, `PATCH`)
`WriteListener`               | `kernel.view`      | `PRE_WRITE`, `POST_WRITE`                   | 32       | Persists changes in the persistence system using the [data persisters](data-persisters.md) (`POST`, `PUT`, `PATCH`, `DELETE`)
`SerializeListener`           | `kernel.view`      | `PRE_SERIALIZE`, `POST_SERIALIZE`           | 16       | Serializes the PHP entity in string [according to the request format](content-negotiation.md)
`RespondListener`             | `kernel.view`      | `PRE_RESPOND`, `POST_RESPOND`               | 8        | Transforms serialized to a `Symfony\Component\HttpFoundation\Response` instance
`AddLinkHeaderListener`       | `kernel.response`  | None                                        | 0        | Adds a `Link` HTTP header pointing to the Hydra documentation
`ValidationExceptionListener` | `kernel.exception` | None                                        | 0        | Serializes validation exceptions in the Hydra format
`ExceptionListener`           | `kernel.exception` | None                                        | -96      | Serializes PHP exceptions in the Hydra format (including the stack trace in debug mode)

Some of these built-in listeners can be enabled/disabled by setting operation attributes:

Attribute     | Type   | Default | Description
--------------|--------|---------|-------------
`read`        | `bool` | `true`  | Enables or disables `ReadListener`
`deserialize` | `bool` | `true`  | Enables or disables `DeserializeListener`
`validate`    | `bool` | `true`  | Enables or disables `ValidateListener`
`write`       | `bool` | `true`  | Enables or disables `WriteListener`
`serialize`   | `bool` | `true`  | Enables or disables `SerializeListener`

Some of these built-in listeners can be enabled/disabled by setting request attributes (for instance in the [`defaults`
attribute of an operation](operations.md#recommended-method)):

Attribute      | Type   | Default | Description
---------------|--------|---------|-------------
`_api_receive` | `bool` | `true`  | Enables or disables `ReadListener`, `DeserializeListener`, `ValidateListener`
`_api_respond` | `bool` | `true`  | Enables or disables `SerializeListener`, `RespondListener`
`_api_persist` | `bool` | `true`  | Enables or disables `WriteListener`

## Custom Event Listeners

Registering your own event listeners to add extra logic is convenient.

The [`ApiPlatform\Core\EventListener\EventPriorities`](https://github.com/api-platform/core/blob/main/src/EventListener/EventPriorities.php) class comes with a convenient set of class constants corresponding to commonly used priorities:

Constant           | Event             | Priority |
-------------------|-------------------|----------|
`PRE_READ`         | `kernel.request`  | 5        |
`POST_READ`        | `kernel.request`  | 3        |
`PRE_DESERIALIZE`  | `kernel.request`  | 3        |
`POST_DESERIALIZE` | `kernel.request`  | 1        |
`PRE_VALIDATE`     | `kernel.view`     | 65       |
`POST_VALIDATE`    | `kernel.view`     | 63       |
`PRE_WRITE`        | `kernel.view`     | 33       |
`POST_WRITE`       | `kernel.view`     | 31       |
`PRE_SERIALIZE`    | `kernel.view`     | 17       |
`POST_SERIALIZE`   | `kernel.view`     | 15       |
`PRE_RESPOND`      | `kernel.view`     | 9        |
`POST_RESPOND`     | `kernel.response` | 0        |

In the following example, we will send a mail each time a new book is created using the API:

```php
<?php
// api/src/EventSubscriber/BookMailSubscriber.php

namespace App\EventSubscriber;

use ApiPlatform\Core\EventListener\EventPriorities;
use App\Entity\Book;
use Symfony\Component\EventDispatcher\EventSubscriberInterface;
use Symfony\Component\HttpFoundation\Request;
use Symfony\Component\HttpKernel\Event\ViewEvent;
use Symfony\Component\HttpKernel\KernelEvents;

final class BookMailSubscriber implements EventSubscriberInterface
{
    private $mailer;

    public function __construct(\Swift_Mailer $mailer)
    {
        $this->mailer = $mailer;
    }

    public static function getSubscribedEvents()
    {
        return [
            KernelEvents::VIEW => ['sendMail', EventPriorities::POST_WRITE],
        ];
    }

    public function sendMail(ViewEvent $event): void
    {
        $book = $event->getControllerResult();
        $method = $event->getRequest()->getMethod();

        if (!$book instanceof Book || Request::METHOD_POST !== $method) {
            return;
        }

        $message = (new \Swift_Message('A new book has been added'))
            ->setFrom('system@example.com')
            ->setTo('contact@les-tilleuls.coop')
            ->setBody(sprintf('The book #%d has been added.', $book->getId()));

        $this->mailer->send($message);
    }
}
```

If you use the official API Platform distribution, creating the previous class is enough. The Symfony DependencyInjection
component will automatically register this subscriber as a service and will inject its dependencies thanks to the [autowiring feature](https://symfony.com/doc/current/service_container/autowiring.html).

Alternatively, [the subscriber must be registered manually](https://symfony.com/doc/current/components/event_dispatcher.html#connecting-listeners).

# The Event System

API Platform Core implements the [Action-Domain-Responder](https://github.com/pmjones/adr) pattern. This implementation
is covered in depth in the [Creating custom operations and controllers](operations.md#creating-custom-operations-and-controllers)
chapter.

Basically, API Platform Core execute an action class that will return an entity or a collection of entities. Then a series
of event listeners are executed which validate the data, persist it in database, serialize it (typically in a JSON-LD document)
and create an HTTP response that will be sent to the client.

To do so, API Platform Core leverages [events triggered by the Symfony HTTP Kernel](https://symfony.com/doc/current/reference/events.html#kernel-events).
You can also hook your own code to those events. They are handy and powerful extension points available at all points
of the request lifecycle.

In the following example, we will send a mail each time a new book is created using the API:

```php
<?php
// api/src/EventSubscriber/BookMailSubscriber.php

namespace App\EventSubscriber;

use ApiPlatform\Core\EventListener\EventPriorities;
use App\Entity\Book;
use Symfony\Component\EventDispatcher\EventSubscriberInterface;
use Symfony\Component\HttpFoundation\Request;
use Symfony\Component\HttpKernel\Event\GetResponseForControllerResultEvent;
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

    public function sendMail(GetResponseForControllerResultEvent $event)
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

If you use the official API Platform distribution, creating the previous class is enough. The Symfony Dependency Injection
component will automatically register this subscriber as a service and will inject its dependencies thanks to the [autowiring
feature](http://symfony.com/doc/current/components/dependency_injection/autowiring.html).

Alternatively, [the subscriber must be registered manually](http://symfony.com/doc/current/components/http_kernel/introduction.html#creating-an-event-listener).

[Doctrine events](http://doctrine-orm.readthedocs.org/en/latest/reference/events.html#reference-events-lifecycle-events)
are also available (if you use it) if you want to hook at the object lifecycle events.

Built-in event listeners are:

Name                          | Event              | Pre & Post hooks                     | Priority | Description
------------------------------|--------------------|--------------------------------------|----------|--------------------------------------------------------------------------------------------------------------------------
`AddFormatListener`           | `kernel.request`   | None                                 | 7        | guess the best response format ([content negotiation](content-negotiation.md))
`ReadListener`                | `kernel.request`   | `PRE_READ`, `POST_READ`              | 4        | retrieve data from the persistence system using the [data providers](data-providers.md) (`GET`, `PUT`, `DELETE`)
`DeserializeListener`         | `kernel.request`   | `PRE_DESERIALIZE`, `POST_DESERIALIZE`| 2        | deserialize data into a PHP entity (`GET`, `POST`, `DELETE`); update the entity retrieved using the data provider (`PUT`)
`ValidateListener`            | `kernel.view`      | `PRE_VALIDATE`, `POST_VALIDATE`      | 64       | [validate data](validation.md) (`POST`, `PUT`)
`WriteListener`               | `kernel.view`      | `PRE_WRITE`, `POST_WRITE`            | 32       | persist changes in the persistence system using the [data persisters](data-persisters.md) (`POST`, `PUT`, `DELETE`)
`SerializeListener`           | `kernel.view`      | `PRE_SERIALIZE`, `POST_SERIALIZE`    | 16       | serialize the PHP entity in string [according to the request format](content-negotiation.md)
`RespondListener`             | `kernel.view`      | `PRE_RESPOND`, `POST_RESPOND`        | 8        | transform serialized to a `Symfony\Component\HttpFoundation\Response` instance
`AddLinkHeaderListener`       | `kernel.response`  | None                                 | 0        | add a `Link` HTTP header pointing to the Hydra documentation
`ValidationExceptionListener` | `kernel.exception` | None                                 | 0        | serialize validation exceptions in the Hydra format
`ExceptionListener`           | `kernel.exception` | None                                 | -96      | serialize PHP exceptions in the Hydra format (including the stack trace in debug mode)

Those built-in listeners are always executed for routes managed by API Platform. Registering your own event listeners to
add extra logic is convenient.

The [`ApiPlatform\Core\EventListener\EventPriorities`](https://github.com/api-platform/core/blob/master/src/EventListener/EventPriorities.php) class comes with a convenient set of class's constants corresponding to commonly used priorities:

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

Some of those built-in listeners can be enabled/disabled by setting request attributes ([for instance in the `defaults` 
attribute of an operation](operations.md#recommended-method)):

Listener              | Parameter      | Values         | Default | Description                            |
----------------------|----------------|----------------|---------|----------------------------------------|
`ReadListener`        | `_api_receive` | `true`/`false` | `true`  | set to `false` to disable the listener |
`DeserializeListener` | `_api_receive` | `true`/`false` | `true`  | set to `false` to disable the listener |
`ValidateListener`    | `_api_receive` | `true`/`false` | `true`  | set to `false` to disable the listener |

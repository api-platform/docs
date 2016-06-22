# The event system

API Platform Core implements the [Action-Domain-Responder](https://github.com/pmjones/adr) pattern. This implementation
is covered in depth in the [Creating custom operations and controllers](operations.md#creating-custom-operations-and-controllers]
chapter.

Basically, API Platform Core execute an action class that will return an entity or a collection of entity. Then a series
of event listeners are executed which validate the data, persist it in database, serialize it (typically in a JSON-LD document)
and create an HTTP response that will be sent to the client.

To do so, API Platform Core leverages [events triggered by the Symfony HTTP Kernel](https://symfony.com/doc/current/reference/events.html#kernel-events).
You can also hook your own code to those events. They are very handy and powerful extension points available at all points
of the request lifecycle.

Built-in event listeners are:

Event               | Priority | Description
--------------------|----------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
`kernel.request`    | 0        | guess the best response format ([content negotiation](content-negotiation.md))
`kernel.controller` | 0        | execute built-in actions that retrieve the data from the persistence system using the [data providers](data-providers.md) (`GET`, `PUT`, `DELETE`), deserializes the request (`POST`, `PUT`) and returns an entity (`GET`, `POST`, `PUT`, `DELETE`), an array of entities or a paged collection (`GET` on a collection)
`kernel.view`       | 20       | [validate data](validation.md) (`POST`, `PUT`)
`kernel.view`       | 10       | persist data (`POST`, `PUT`, `DELETE`)
`kernel.view`       | 0        | serialize data and transform them in a `Symfony\Component\HttpFoundation\Response` instance

Those built-in listeners are always executed for routes managed by API Platform. Registering your own event listeners to
add extra logic is convenient.

In the following example, we will send a mail each time a new book is created using the API:

```php
// src/AppBundle/EventSubscriber/BookMailSubscriber.php

namespace AppBundle\EventSubscriber;

use AppBundle\Entity\Book;
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
            KernelEvents::VIEW => [['sendMail', 5]],
        ];
    }

    public function sendMail(GetResponseForControllerResultEvent $event)
    {
        $book = $event->getControllerResult();
        $method = $event->getRequest()->getMethod();

        if (!$book instanceof Book || Request::METHOD_POST !== $method) {
            return;
        }

        $message = \Swift_Message::newInstance()
            ->setSubject('A new book has been added')
            ->setFrom('system@example.com')
            ->setTo('contact@les-tilleuls.coop')
            ->setBody(sprintf('The book #%d has been added.', $book->getId()));

        $this->mailer->send($message);
    }
}
```

If you use the standard edition of API Platform, creating the previous class is enough. [DunglasActionBundle](https://github.com/dunglas/DunglasActionBundle)
(installed by default) will automatically register this subscriber as a service and will inject its dependencies using [the
autowiring feature of the Symfony Dependency Injection Container](http://symfony.com/doc/current/components/dependency_injection/autowiring.html).

If you don't have DunglasActionBundle installed, [the subscriber must be registered manually](http://symfony.com/doc/current/components/http_kernel/introduction.html#creating-an-event-listener).

[Doctrine events](http://doctrine-orm.readthedocs.org/en/latest/reference/events.html#reference-events-lifecycle-events)
are also available (if you use it) if you want to hook at the object lifecycle events.

Previous chapter: [Validation](validation.md)<br>
Next chapter: [Resources](resources.md)

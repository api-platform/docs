# Errors Handling

API Platform comes with a powerful error system. It handles expected (such as faulty JSON documents sent by the
client or validation errors) as well as unexpected errors (PHP exceptions and errors).
API Platform automatically sends the appropriate HTTP status code to the client: `400` for expected errors, `500` for
unexpected ones. It also provides a description of the error in [the Hydra error format](https://www.hydra-cg.com/spec/latest/core/#description-of-http-status-codes-and-errors)
or in the format described in the [RFC 7807](https://tools.ietf.org/html/rfc7807), depending of the format selected during the [content negotiation](content-negotiation.md).

## Converting PHP Exceptions to HTTP Errors

The framework also allows you to configure the HTTP status code sent to the clients when custom exceptions are thrown
on an API Platform resource operation.

In the following example, we throw a domain exception from the business layer of the application and
configure API Platform to convert it to a `404 Not Found` error:

```php
<?php
// api/src/Exception/ProductNotFoundException.php

namespace App\Exception;

final class ProductNotFoundException extends \Exception
{
}
```

```php
<?php
// api/src/EventSubscriber/ProductManager.php

namespace App\EventSubscriber;

use ApiPlatform\Core\EventListener\EventPriorities;
use App\Entity\Product;
use App\Exception\ProductNotFoundException;
use Symfony\Component\EventDispatcher\EventSubscriberInterface;
use Symfony\Component\HttpFoundation\Request;
use Symfony\Component\HttpKernel\Event\ViewEvent;
use Symfony\Component\HttpKernel\KernelEvents;

final class ProductManager implements EventSubscriberInterface
{
    public static function getSubscribedEvents(): array
    {
        return [
            KernelEvents::VIEW => ['checkProductAvailability', EventPriorities::PRE_VALIDATE],
        ];
    }

    public function checkProductAvailability(ViewEvent $event): void
    {
        $product = $event->getControllerResult();
        if (!$product instanceof Product || !$event->getRequest()->isMethodSafe(false)) {
            return;
        }

        if (!$product->isPubliclyAvailable()) {
            // Using internal codes for a better understanding of what's going on
            throw new ProductNotFoundException(sprintf('The product "%s" does not exist.', $product->getId()));
        }
    }
}
```

If you use the standard distribution of API Platform, this event listener will be automatically registered. If you use a
custom installation, [learn how to register listeners](events.md#custom-event-listeners).

Then, configure the framework to catch `App\Exception\ProductNotFoundException` exceptions and convert them into `404`
errors:

```yaml
# config/packages/api_platform.yaml
api_platform:
    # ...
    exception_to_status:
        # The 4 following handlers are registered by default, keep those lines to prevent unexpected side effects
        Symfony\Component\Serializer\Exception\ExceptionInterface: 400 # Use a raw status code (recommended)
        ApiPlatform\Core\Exception\InvalidArgumentException: !php/const Symfony\Component\HttpFoundation\Response::HTTP_BAD_REQUEST
        ApiPlatform\Core\Exception\FilterValidationException: 400
        Doctrine\ORM\OptimisticLockException: 409

        # Validation exception
        ApiPlatform\Core\Bridge\Symfony\Validator\Exception\ValidationException: !php/const Symfony\Component\HttpFoundation\Response::HTTP_UNPROCESSABLE_ENTITY

        # Custom mapping
        App\Exception\ProductNotFoundException: 404 # Here is the handler for our custom exception
```

Any type of `Exception` can be thrown, API Platform will convert it to a Symfony's `HttpException` (note that it means the exception will be flattened and lose all of its custom properties). The framework also takes
care of serializing the error description according to the request format. For instance, if the API should respond in JSON-LD,
the error will be returned in this format as well:

`GET /products/1234`

```json
{
  "@context": "/contexts/Error",
  "@type": "Error",
  "hydra:title": "An error occurred",
  "hydra:description": "The product \"1234\" does not exist."
}
```

## Message Scope

Depending on the status code you use, the message may be replaced with a generic one in production to avoid leaking unwanted information.
If your status code is >= 500 and < 600, the exception message will only be displayed in debug mode (dev and test). In production, a generic message matching the status code provided will be shown instead. If you are using an unofficial HTTP code, a general message will be displayed.

In any other cases, your exception message will be sent to end users.

## Fine-grained Configuration

The `exception_to_status` configuration can be set on resources and operations:

```php
<?php
// api/src/Entity/Book.php
namespace App\Entity;

use ApiPlatform\Core\Annotation\ApiResource;
use App\Exception\ProductWasRemovedException;
use App\Exception\ProductNotFoundException;

#[ApiResource(
    itemOperations: [
        'get' => [
            'exception_to_status' => [
                ProductWasRemovedException::class => 410,
            ],
        ],
    ],
    exceptionToStatus: [
        ProductNotFoundException::class => 404,
    ]
)]
class Book
{
    // ...
}
```

Exceptions mappings defined on operations take precedence over mappings defined on resources, which take precedence over
the global config.

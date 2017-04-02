# Error Handling

API Platform comes with a powerful error system. It handles excepted (such as faulty JSON documents sent by the
client or validation errors) as well as unexpected errors (PHP exceptions and errors).
API Platform automatically send the appropriate HTTP status code to the client: `400` for expected errors, `500` for
unexpected ones. It also provides a description of the respecting [the Hydra specification](http://www.hydra-cg.com/spec/latest/core/#description-of-http-status-codes-and-errors)
or the [RFC 7807](https://tools.ietf.org/html/rfc7807) depending of the format selected during the [content negotiation](content-negotiation.md).

## Converting PHP Exceptions to HTTP Errors

The framework also allows to configure the HTTP status code sent to the clients when custom exceptions are thrown.

In the following example, we will throw explain to throw a domain exception from the business layer of the application and
configure API Platform to convert it to a `404 Not Found` error. Let's create a this domain exception and the service throwing
it:

```php
<?php
// src/AppBundle/Exception/ProductNotFoundException.php

namespace AppBundle\Exception;

final class ProductNotFoundException extends \Exception
{
}
```

```php
<?php
// src/AppBundle/EventSubscriber/CartManager.php

namespace AppBundle\EventSubscriber;

use ApiPlatform\Core\EventListener\EventPriorities;
use AppBundle\Entity\Product;
use AppBundle\Exception\ProductNotFoundException;
use Symfony\Component\EventDispatcher\EventSubscriberInterface;
use Symfony\Component\HttpFoundation\Request;
use Symfony\Component\HttpKernel\Event\GetResponseForControllerResultEvent;
use Symfony\Component\HttpKernel\KernelEvents;

final class ProductManager implements EventSubscriberInterface
{
    public static function getSubscribedEvents(): array
    {
        return [
            KernelEvents::REQUEST => ['checkProductAvailability', EventPriorities::POST_DESERIALIZE],
        ];
    }

    public function checkProductAvailability(GetResponseForControllerResultEvent $event): void
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

If you use the standard distribution of API Platform, the event listener will be automatically registered. If you use a
custom installation, [learn how to register listeners](events.md).

Then, configure the framework to catch `AppBundle\Exception\ProductNotFoundException` exceptions and convert them in `404`
errors:

```yaml
# app/config/config.yml
api_platform:
    # ...
    exception_to_status:
        # The 2 following handlers are registered by default, keep those lines to prevent unexpected side effects
        Symfony\Component\Serializer\Exception\ExceptionInterface: 400 # Use a raw status code (recommended)
        ApiPlatform\Core\Exception\InvalidArgumentException: 'HTTP_BAD_REQUEST' # Or a `Symfony\Component\HttpFoundation\Response`'s constant

        AppBundle\Exception\ProductNotFoundException: 404 # Here is the handler for our custom exception
```

Any type of `Exception` can be thrown, API Platform will convert it to a Symfony's `HttpException`. The framework also takes
care to serialize the error description according to the request format. For instance, if the API should respond in JSON-LD,
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

Previous chapter: [Pagination](pagination.md)

Next chapter: [The Event System](events.md)

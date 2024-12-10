# Errors Handling

API Platform comes with a powerful error system. It handles expected (such as faulty JSON documents sent by the
client or validation errors) as well as unexpected errors (PHP exceptions and errors).
API Platform automatically sends the appropriate HTTP status code to the client: `400` for expected errors, `500` for
unexpected ones. It also provides a description of the error in [the Hydra error format](https://www.hydra-cg.com/spec/latest/core/#description-of-http-status-codes-and-errors)
or in the format described in the [RFC 7807](https://tools.ietf.org/html/rfc7807), depending on the format selected during the [content negotiation](content-negotiation.md).

## Backward compatibility with < 3.1

Use the following configuration:

```yaml
api_platform:
  defaults:
    extra_properties:
      rfc_7807_compliant_errors: false
```

This can also be configured on an `ApiResource` or in an `HttpOperation`, for example:

```php
#[ApiResource(extraProperties: ['rfc_7807_compliant_errors' => false])
```

## Exception status code decision

There are many ways of configuring the exception status code we recommend reading the guides on how to use an
[Error Provider](https://api-platform.com/docs/guides/error-provider/) or create an [Error Resource](https://api-platform.com/docs/guides/error-resource/).

The decision works like this, if you are using API Platform with Symfony:

1. We look at `exception_to_status` and take one if there's a match
2. If your exception is a `Symfony\Component\HttpKernel\Exception\HttpExceptionInterface` we get its status.
3. If the exception is a `ApiPlatform\Metadata\Exception\ProblemExceptionInterface` and there is a status we use it
4. Same for `ApiPlatform\Metadata\Exception\HttpExceptionInterface`
5. Use defaults for the following exceptions:
    - `Symfony\Component\HttpFoundation\Exception\RequestExceptionInterface` => 400
    - `ApiPlatform\Symfony\Validator\Exception\ValidationException` => 422
6. The status defined on an `ErrorResource`
7. 500 is the fallback

And like this, if you are using API Platform with Laravel:

1. Check an `exception_to_status` array and use its value if a match is found.
2. If the exception implements `Illuminate\Contracts\Http\Exception\HttpResponseException`, retrieve its HTTP status.
3. If the exception implements `App\Contracts\Exceptions\ProblemExceptionInterface` and a status is defined, use it.
4. Similarly, check for `App\Contracts\Exceptions\HttpExceptionInterface`.
5. Use defaults for the following exceptions:
    - `Illuminate\Http\Exceptions\HttpResponseException` => 400
    - `ApiPlatform\Symfony\Validator\Exception\ValidationException` => 422
6. The status defined on an `ErrorResource`
7. Fallback to 500.

## Exception to status

The framework also allows you to configure the HTTP status code sent to the clients when custom exceptions are thrown
on an API Platform resource operation.

In the following example, we throw a domain exception from the business layer of the application and
configure API Platform to convert it to a `404 Not Found` error:

```php
<?php
// api/src/Exception/ProductNotFoundException.php with Symfony or app/Exception/ProductNotFoundException.php with Laravel
namespace App\Exception;

final class ProductNotFoundException extends \Exception
{
    // ...
}
```

```php
<?php
// api/src/EventSubscriber/ProductManager.php with Symfony or app/EventSubscriber/ProductManager.php with Laravel

namespace App\EventSubscriber;

use ApiPlatform\EventListener\EventPriorities;
use App\ApiResource\Product;
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
custom installation, [learn how to extend API Platform](extending.md).

Then, configure the framework to catch `App\Exception\ProductNotFoundException` exceptions and convert them into `404`
errors:

### Exception to status Configuration using Symfony

```yaml
# config/packages/api_platform.yaml
api_platform:
  # ...
  exception_to_status:
    # The 4 following handlers are registered by default, keep those lines to prevent unexpected side effects
    Symfony\Component\Serializer\Exception\ExceptionInterface: 400 # Use a raw status code (recommended)
    ApiPlatform\Exception\InvalidArgumentException: !php/const Symfony\Component\HttpFoundation\Response::HTTP_BAD_REQUEST
    ApiPlatform\ParameterValidator\Exception\ValidationExceptionInterface: 400
    Doctrine\ORM\OptimisticLockException: 409

    # Validation exception
    ApiPlatform\Validator\Exception\ValidationException: !php/const Symfony\Component\HttpFoundation\Response::HTTP_UNPROCESSABLE_ENTITY

    # Custom mapping
    App\Exception\ProductNotFoundException: 404 # Here is the handler for our custom exception
```

### Exception to status Configuration using Laravel

```php
<?php
// config/api-platform.php
return [
    // ....
    'exception_to_status' => [
        // The 3 following handlers are registered by default, keep those lines to prevent unexpected side effects
        Symfony\Component\Serializer\Exception\ExceptionInterface::class => 400,
        ApiPlatform\Exception\InvalidArgumentException::class => Illuminate\Http\Response::HTTP_BAD_REQUEST,
        ApiPlatform\ParameterValidator\Exception\ValidationExceptionInterface => 400,

        //Validation exception
        ApiPlatform\Validator\Exception\ValidationException::class => Illuminate\Http\Response::HTTP_UNPROCESSABLE_ENTITY,
        
        //Custom mapping
        App\Exception\ProductNotFoundException::class => 404 // Here is the handler for our custom exception
    ],
];
```

Any type of `Exception` can be thrown, API Platform will convert it to a Symfony's `HttpException` (note that it means
the exception will be flattened and lose all of its custom properties). The framework also takes care of serializing the
error description according to the request format. For instance, if the API should respond in JSON-LD, the error will be
returned in this format as well:

`GET /products/1234`

```json
{
  "@context": "/contexts/Error",
  "@type": "Error",
  "title": "An error occurred",
  "description": "The product \"1234\" does not exist."
}
```

### Message Scope

Depending on the status code you use, the message may be replaced with a generic one in production to avoid leaking unwanted information.
If your status code is >= 500 and < 600, the exception message will only be displayed in debug mode (dev and test).
In production, a generic message matching the status code provided will be shown instead. If you are using an unofficial
HTTP code, a general message will be displayed.

In any other cases, your exception message will be sent to end users.

### Fine-grained Configuration

The `exceptionToStatus` configuration can be set on resources and operations:

```php
<?php
// api/src/ApiResource/Book.php with Symfony or app/ApiResource/Book.php with Laravel
namespace App\ApiResource;

use ApiPlatform\Metadata\ApiResource;
use ApiPlatform\Metadata\Get;
use ApiPlatform\Metadata\GetCollection;
use ApiPlatform\Metadata\Post;
use App\Exception\ProductWasRemovedException;
use App\Exception\ProductNotFoundException;

#[ApiResource(
    exceptionToStatus: [ProductNotFoundException::class => 404]
    operations: [
        new Get(exceptionToStatus: [ProductWasRemovedException::class => 410]),
        new GetCollection(),
        new Post()
    ]
)]
class Book
{
    // ...
}
```

Exceptions mappings defined on operations take precedence over mappings defined on resources, which take precedence over
the global config.

## Control your exceptions

With `rfc_7807_compliant_errors` a few things happen. First Hydra exception are compatible with the JSON Problem specification.
Default exception that are handled by API Platform in JSON will be returned as `application/problem+json`.

To customize the API Platform response, replace the `api_platform.state.error_provider` with your own provider:

```php
<?php

namespace App\State;

use ApiPlatform\Metadata\Operation;
use ApiPlatform\State\ApiResource\Error;
use ApiPlatform\State\ProviderInterface;
use Symfony\Component\DependencyInjection\Attribute\AsAlias;
use Symfony\Component\DependencyInjection\Attribute\AsTaggedItem;

#[AsAlias('api_platform.state.error_provider')]
#[AsTaggedItem('api_platform.state.error_provider')]
final class ErrorProvider implements ProviderInterface
{
    public function provide(Operation $operation, array $uriVariables = [], array $context = []): object|array|null
    {
        $request = $context['request'];
        $format = $request->getRequestFormat();
        $exception = $request->attributes->get('exception');

        /** @var \ApiPlatform\Metadata\HttpOperation $operation */
        $status = $operation->getStatus() ?? 500;
        // You don't have to use this, you can use a Response, an array or any object (preferably a resource that API Platform can handle).
        $error = Error::createFromException($exception, $status);

        // care about hiding information as this can be a security leak
        if ($status >= 500) {
            $error->setDetail('Something went wrong');
        }

        return $error;
    }
}
```

```yaml
# The YAML syntax is only supported for Symfony
api_platform.state.error_provider:
  class: 'App\State\ErrorProvider'
  tags:
    - key: 'api_platform.state.error_provider'
      name: 'api_platform.state_provider'
```

Note that our validation exception have their own error provider at:

```yaml
# The YAML syntax is only supported for Symfony
api_platform.validator.state.error_provider:
  tags:
    - key: 'api_platform.validator.state.error_provider'
      name: 'api_platform.state_provider'
```

## Domain exceptions

Another way of having full control over domain exceptions is to create your own Error resource:

```php
<?php

namespace App\ApiResource;

use ApiPlatform\Metadata\ErrorResource;
use ApiPlatform\Metadata\Exception\ProblemExceptionInterface;

#[ErrorResource]
class Error extends \Exception implements ProblemExceptionInterface
{
    public function getType(): string
    {
        return 'teapot';
    }

    public function getTitle(): ?string
    {
        return null;
    }

    public function getStatus(): ?int
    {
        return 418;
    }

    public function getDetail(): ?string
    {
        return 'I am teapot';
    }

    public function getInstance(): ?string
    {
        return null;
    }
}
```

We recommend using the `\ApiPlatform\Metadata\Exception\ProblemExceptionInterface` and the
`\ApiPlatform\Metadata\Exception\HttpExceptionInterface`. For security reasons we add: `normalizationContext: ['ignored_attributes'
=> ['trace', 'file', 'line', 'code', 'message', 'traceAsString']]` because you usually don't want these. You can override
this context value if you want.

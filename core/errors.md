# Error Handling

API Platform Core allows to customize the HTTP status code sent to the clients when exceptions are thrown.

```yaml
# app/config/config.yml

 api_platform:
# Map exceptions to HTTP status codes using the `exception_to_status` configuration key
    exception_to_status:
        # The 2 following exceptions are handled by default
        Symfony\Component\Serializer\Exception\ExceptionInterface: 400 # Use a raw status code (recommended)
        ApiPlatform\Core\Exception\InvalidArgumentException: 'HTTP_BAD_REQUEST' # Or with a constant of `Symfony\Component\HttpFoundation\Response`

        AppBundle\Exception\ProductNotFoundException: 404 # Custom exceptions can easily be handled
```

As in any php application, your exceptions have to extends the \Exception class or any of it's children.

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
    private const DOESNOTEXISTS = 51;
    private const DEACTIVATED = 52;
    private const OUTOFSTOCK = 53;

    public static function getSubscribedEvents()
    {
        return [
            KernelEvents::REQUEST => [['checkProductAvailability', EventPriorities::POST_DESERIALIZE]],
        ];
    }

    public function checkProductAvailability(GetResponseForControllerResultEvent $event)
    {
        $product = $event->getControllerResult();
        $method = $event->getRequest()->getMethod();

        if (!$product instanceof Product || Request::METHOD_GET !== $method) {
            return;
        }

        if (!$product->getVirtualStock()) {
            // Using internal codes for a better understanding of what's going on
            throw new ProductNotFoundException(self::OUTOFSTOCK);
        }
    }
}
```

The exception doesn't have to be a Symfony's `HttpException`. Any type of `Exception` can be thrown. The best part is that API Platform already takes care of how the error is handled and returned. For instance, if the API is configured to respond in JSON-LD, the error will be returned in this format as well.

```json
{
  "@context": "/contexts/Error",
  "@type": "Error",
  "hydra:title": "An error occurred",
  "hydra:description": "53"
}
```

Is what you get, with an HTTP status code 404 as defined in the configuration.

## Validation errors

API Platform does handle the validation errors responses for you. You can define a Symfony supported constraint, or a custom constraint upon any `ApiResource` or it's properties.

```php
<?php

// src/AppBundle/Entity/Product.php

namespace AppBundle\Entity;

use ApiPlatform\Core\Annotation\ApiResource;
use Doctrine\ORM\Mapping as ORM;
use Symfony\Component\Validator\Constraints as Assert;
use AppBundle\Validator\Constraints as AppAssert;

/**
 *
 * @ApiResource
 * @ORM\Entity
 */
class Product
{
    /**
     * @var int The id of this product.
     *
     * @ORM\Id
     * @ORM\GeneratedValue
     * @ORM\Column(type="integer")
     */
    private $id;

    /**
     * @var string The name of the product
     *
     * @ORM\Column
     * @Assert\NotBlank
     */
    private name;

    /**
     * @var ProductProperty[] Describe the product
     *
     * @ORM\Column(type="array")
     * @AppAssert\MinimalProperties
     */
    private $properties;
}
```

```php
<?php

// src/AppBundle/Validator/Constraints/MinimalProperties.php

namespace AppBundle\Validator\Constraints;

use Symfony\Component\Validator\Constraint;

/**
 * @Annotation
 */
class MinimalProperties extends Constraint
{
    public $message = 'The product must have the minimal properties required (description, price)';

    public function validatedBy()
    {
        return get_class($this).'Validator';
    }
}
```

```php
<?php

// src/AppBundle/Validator/Constraints/MinimalPropertiesValidator.php

namespace AppBundle\Validator\Constraints;

use Symfony\Component\Validator\Constraint;
use Symfony\Component\Validator\ConstraintValidator;

/**
 * @Annotation
 */
final class MinimalPropertiesValidator extends ConstraintValidator
{
    public function validate($value, Constraint $constraint)
    {
        if (!in_array('description', $value) || !in_array('price', $value)) {
            $this->context->buildViolation($constraint->message)
                ->addViolation();
        }
    }
}
```

API Platform will handle the error returned and adapt it's format according to the API configuration. If you did configured it to respond in JSON-LD. Your response would looks like:

```json
{
  "@context": "/contexts/ConstraintViolationList",
  "@type": "ConstraintViolationList",
  "hydra:title": "An error occurred",
  "hydra:description": "properties: The product must have the minimal properties required (description, price)",
  "violations": [
    {
      "propertyPath": "properties",
      "message": "The product must have the minimal properties required (description, price)"
    }
  ]
}
```

Previous chapter: [Pagination](pagination.md)

Next chapter: [The Event System](events.md)

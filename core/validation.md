# Validation

API Platform take care of validating data sent to the API by the client (usually user data entered through forms).
By default, the framework relies on [the powerful Symfony Validator Component](http://symfony.com/doc/current/validation.html)
for this task, but you can replace it by your preferred validation library such as [the PHP filter extension](http://php.net/manual/en/intro.filter.php)
if you want to.

## Validating Submitted Data

Validating submitted data is simple as adding [Symfony's builtin constraints](http://symfony.com/doc/current/reference/constraints.html)
or [custom constraints](http://symfony.com/doc/current/validation/custom_constraint.html) directly in classes marked with
the `@ApiResource` annotation:

```php
<?php
// src/AppBundle/Entity/Product.php

namespace AppBundle\Entity;

use ApiPlatform\Core\Annotation\ApiResource;
use Doctrine\ORM\Mapping as ORM;
use Symfony\Component\Validator\Constraints as Assert; // Symfony's builtin constraints
use AppBundle\Validator\Constraints\MinimalProperties; // A custom constraint

/**
 * A product.
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
     * @Assert\NotBlank
     * @ORM\Column
     */
    private name;

    /**
     * @var string[] Describe the product
     *
     * @MinimalProperties
     * @ORM\Column(type="json")
     */
    private $properties;

    // Getters and setters...
}
```

Here is a custom constraint and the related validator:

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
    public $message = 'The product must have the minimal properties required ("description", "price")';
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
    public function validate($value, Constraint $constraint): void
    {
        if (!array_diff(['description', 'price'], $value)) {
            $this->context->buildViolation($constraint->message)->addViolation();
        }
    }
}
```

If the data submitted by the client is invalid, the HTTP status code will be set to `400 Bad Request` and the response's
body will contain the list of violations serialized in a format compliant with the requested one. For instance, a validation
error will look like the following if the requested format is JSON-LD (the default):

```json
{
  "@context": "/contexts/ConstraintViolationList",
  "@type": "ConstraintViolationList",
  "hydra:title": "An error occurred",
  "hydra:description": "properties: The product must have the minimal properties required (\"description\", \"price\")",
  "violations": [
    {
      "propertyPath": "properties",
      "message": "The product must have the minimal properties required (\"description\", \"price\")"
    }
  ]
}
```

Take a look at the [Errors Handling guide](errors.md) to learn how API Platform converts PHP exceptions like validation
errors to HTTP errors.

## Using Validation Groups

Without specific configuration, the default validation group is always used, but this behavior is customizable: the framework
is able to leverage Symfony's [validation groups](http://symfony.com/doc/current/book/validation.html#validation-groups).

You can configure the groups you want to use when the validation occurs directly through the `ApiResource` annotation:

```php
<?php

// src/AppBundle/Entity/Book.php

use ApiPlatform\Core\Annotation\ApiResource;
use Symfony\Component\Validator\Constraints as Assert;

/**
 * @ApiResource(attributes={"validation_groups"={"a", "b"}})
 * ...
 */
class Book
{
    /**
     * @Assert\NotBlank(groups={"a"})
     */
    private $name;

    /**
     * @Assert\NotNull(groups={"b"})
     */
    private $author;

    // ...
}
```

With the previous configuration, the validations groups `a` and `b` will be used when validation is performed.

Like for [serialization groups](serialization-groups-and-relations.md#using-different-serialization-groups-per-operation),
you can specify validation groups globally or on a per operation basis.

Of course, you can use XML or YAML configuration format instead of annotations if you prefer.

You may also pass in a [group sequence](http://symfony.com/doc/current/validation/sequence_provider.html) in place of
the array of group names.

## Dynamic Validation Groups

If you need to dynamically determine which validation groups to use for an entity in different scenarios, just pass in a
[callable](http://php.net/manual/en/language.types.callable.php). The callback will receive the entity object as its first
argument, and should return an array of group names or a [group sequence](http://symfony.com/doc/current/validation/sequence_provider.html).

In the following example, we use a static method to return the validation groups:

```php
<?php

// src/AppBundle/Entity/Book.php

use ApiPlatform\Core\Annotation\ApiResource;
use Symfony\Component\Validator\Constraints as Assert;

/**
 * @ApiResource(attributes={"validation_groups"={Book::class, "validationGroups"}})
 */
class Book
{
    /**
     * Return dynamic validation groups.
     *
     * @param self $book Contains the instance of Book to validate.
     *
     * @return string[]
     */
    public static function validationGroups(self $book)
    {
        return ['a'];
    }

    /**
     * @Assert\NotBlank(groups={"a"})
     */
    private $name;

    /**
     * @Assert\NotNull(groups={"b"})
     */
    private $author;

    // ...
}
```

Previous chapter: [Serialization Groups and Relations](serialization-groups-and-relations.md)

Next chapter: [Pagination](pagination.md)

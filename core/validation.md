# Validation

API Platform takes care of validating the data sent to the API by the client (usually user data entered through forms).
By default, the framework relies on [the powerful Symfony Validator Component](http://symfony.com/doc/current/validation.html)
for this task, but you can replace it with your preferred validation library such as [the PHP filter extension](https://www.php.net/manual/en/intro.filter.php) if you want to.

<p align="center" class="symfonycasts"><a href="https://symfonycasts.com/screencast/api-platform/validation?cid=apip"><img src="../distribution/images/symfonycasts-player.png" alt="Validation screencast"><br>Watch the Validation screencast</a></p>

## Validating Submitted Data

Validating submitted data is as simple as adding [Symfony's built-in constraints](http://symfony.com/doc/current/reference/constraints.html)
or [custom constraints](http://symfony.com/doc/current/validation/custom_constraint.html) directly in classes marked with
the `@ApiResource` annotation:

```php
<?php
// api/src/Entity/Product.php

namespace App\Entity;

use ApiPlatform\Core\Annotation\ApiResource;
use App\Validator\Constraints\MinimalProperties; // A custom constraint
use Doctrine\ORM\Mapping as ORM;
use Symfony\Component\Validator\Constraints as Assert; // Symfony's built-in constraints

/**
 * A product.
 *
 * @ORM\Entity
 */
#[ApiResource]
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
    public $name;

    /**
     * @var string[] Describe the product
     *
     * @MinimalProperties
     * @ORM\Column(type="json")
     */
    public $properties;

    // Getters and setters...
}
```

And here is the custom `MinimalProperties` constraint and the related validator:

```php
<?php
// api/src/Validator/Constraints/MinimalProperties.php

namespace App\Validator\Constraints;

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
// api/src/Validator/Constraints/MinimalPropertiesValidator.php

namespace App\Validator\Constraints;

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

If the data submitted by the client is invalid, the HTTP status code will be set to `422 Unprocessable Entity` and the response's
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
is able to leverage Symfony's [validation groups](https://symfony.com/doc/current/validation/groups.html).

You can configure the groups you want to use when the validation occurs directly through the `ApiResource` annotation:

```php
<?php
// api/src/Entity/Book.php

use ApiPlatform\Core\Annotation\ApiResource;
use Symfony\Component\Validator\Constraints as Assert;

#[ApiResource(attributes: ['validation_groups' => ['a', 'b']])]
class Book
{
    /**
     * @Assert\NotBlank(groups={"a"})
     */
    public $name;

    /**
     * @Assert\NotNull(groups={"b"})
     */
    public $author;

    // ...
}
```

With the previous configuration, the validation groups `a` and `b` will be used when validation is performed.

Like for [serialization groups](serialization.md#using-different-serialization-groups-per-operation),
you can specify validation groups globally or on a per-operation basis.

Of course, you can use XML or YAML configuration format instead of annotations if you prefer.

You may also pass in a [group sequence](http://symfony.com/doc/current/validation/sequence_provider.html) in place of
the array of group names.

## Using Validation Groups on Operations

You can have different validation for each [operation](operations.md) related to your resource.

```php
<?php
// api/src/Entity/Book.php

use ApiPlatform\Core\Annotation\ApiResource;
use Symfony\Component\Validator\Constraints as Assert;

#[ApiResource(
    collectionOperations: [
        'get',
        'post' => ['validation_groups' => ['Default', 'postValidation']]
    ],
    itemOperations: [
        'delete',
        'get',
        'put' => ['validation_groups' => ['Default', 'putValidation']]
    ]
)]
class Book
{
    /**
     * @Assert\Uuid
     */
    private $id;

    /**
     * @Assert\NotBlank(groups={"postValidation"})
     */
    public $name;

    /**
     * @Assert\NotNull
     * @Assert\Length(
     *     min = 2,
     *     max = 50,
     *     groups={"postValidation"}
     * )
     * @Assert\Length(
     *     min = 2,
     *     max = 70,
     *     groups={"putValidation"}
     * )
     */
    public $author;

    // ...
}
```

With this configuration, there are three validation groups:

`Default` contains the constraints that belong to no other group.

`postValidation` contains the constraints on the name and author (length from 2 to 50) fields only.

`putValidation` contains the constraints on the author (length from 2 to 70) field only.

## Dynamic Validation Groups

If you need to dynamically determine which validation groups to use for an entity in different scenarios, just pass in a
[callable](https://www.php.net/manual/en/language.types.callable.php). The callback will receive the entity object as its first
argument, and should return an array of group names or a [group sequence](http://symfony.com/doc/current/validation/sequence_provider.html).

In the following example, we use a static method to return the validation groups:

```php
<?php
// api/src/Entity/Book.php

use ApiPlatform\Core\Annotation\ApiResource;
use Symfony\Component\Validator\Constraints as Assert;

#[ApiResource(
    attributes: ['validation_groups' => [Book::class, 'validationGroups']]
)]
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
    public $name;

    /**
     * @Assert\NotNull(groups={"b"})
     */
    public $author;

    // ...
}
```

Alternatively, you can use a service to retrieve the groups to use:

```php
<?php
// api/src/Validator/AdminGroupsGenerator.php

namespace App\Validator;

use ApiPlatform\Core\Bridge\Symfony\Validator\ValidationGroupsGeneratorInterface;
use App\Entity\Book;
use Symfony\Component\Security\Core\Authorization\AuthorizationCheckerInterface;

final class AdminGroupsGenerator implements ValidationGroupsGeneratorInterface
{
    private $authorizationChecker;

    public function __construct(AuthorizationCheckerInterface $authorizationChecker)
    {
        $this->authorizationChecker = $authorizationChecker;
    }

    /**
     * {@inheritdoc}
     */
    public function __invoke($book): array
    {
        assert($book instanceof Book);

        return $this->authorizationChecker->isGranted('ROLE_ADMIN', $book) ? ['a', 'b'] : ['a'];
    }
}
```

This class selects the groups to apply based on the role of the current user: if the current user has the `ROLE_ADMIN` role, groups `a` and `b` are returned. In other cases, just `a` is returned.

This class is automatically registered as a service thanks to [the autowiring feature of the Symfony DependencyInjection component](https://symfony.com/doc/current/service_container/autowiring.html).

Then, configure the entity class to use this service to retrieve validation groups:

```php
<?php
// api/src/Entity/Book.php

namespace App\Entity;

use ApiPlatform\Core\Annotation\ApiResource;
use App\Validator\AdminGroupsGenerator;
use Symfony\Component\Validator\Constraints as Assert;

#[ApiResource(attributes: ['validation_groups' => AdminGroupsGenerator::class])
class Book
{
    /**
     * @Assert\NotBlank(groups={"a"})
     */
    public $name;

    /**
     * @Assert\NotNull(groups={"b"})
     */
    public $author;

    // ...
}
```

## Sequential Validation Groups

If you need to specify the order in which your validation groups must be tested against, you can use a [group sequence](http://symfony.com/doc/current/validation/sequence_provider.html).
First, you need to create your sequenced group.

```php
<?php

declare(strict_types=1);

namespace App\Validator;

use Symfony\Component\Validator\Constraints\GroupSequence;

class MySequencedGroup
{
    public function __invoke()
    {
        return new GroupSequence(['first', 'second']); // now, no matter which is first in the class declaration, it will be tested in this order.
    }
}
```

Just creating the class is not enough because Symfony does not see this service as being used. Therefore to prevent the service to be removed, you need to enforce it to be public.

```yaml
# api/config/services.yaml
services:
    App\Validator\MySequencedGroup: ~
        public: true
```

And then, you need to use your class as a validation group.

```php
<?php

namespace App\Entity;

use ApiPlatform\Core\Annotation\ApiResource;
use App\Validator\One; // classic custom constraint
use App\Validator\Two; // classic custom constraint
use App\Validator\MySequencedGroup; // the sequence group to use
use Doctrine\ORM\Mapping as ORM;

/**
 * @ORM\Entity
 */
#[ApiResource(
    collectionOperations: [
      'post' => [
        'validation_groups' => MySequencedGroup::class
      ]
    ]
)]
class Greeting
{
    /**
     * @var int The entity Id
     *
     * @ORM\Id
     * @ORM\GeneratedValue
     * @ORM\Column(type="integer")
     */
    private $id;

    /**
     * @var string A nice person
     *
     * @ORM\Column
     * 
     * I want this "second" validation to be executed after the "first" one even though I wrote them in this order.
     * @One(groups={"second"})
     * @Two(groups={"first"})
     */
    public $name = '';

    public function getId(): int
    {
        return $this->id;
    }
}
```

## Error Levels and Payload Serialization

As stated in the [Symfony documentation](https://symfony.com/doc/current/validation/severity.html), you can use the payload field to define error levels.
You can retrieve the payload field by setting the `serialize_payload_fields` to an empty `array` in the API Platform config:

```yaml
# api/config/packages/api_platform.yaml

api_platform:
    validator:
        serialize_payload_fields: ~
```

Then, the serializer will return all payload values in the error response.

If you want to serialize only some payload fields, define them in the config like this:

```yaml
# api/config/packages/api_platform.yaml

api_platform:
    validator:
        serialize_payload_fields: [ severity, anotherPayloadField ]
```

In this example, only `severity` and `anotherPayloadField` will be serialized.

## Validation on Collection Relations

Use the [Valid](https://symfony.com/doc/current/reference/constraints/Valid.html) constraint.

Note: this is related to the [collection relation denormalization](./serialization.md#collection-relation).
You may have an issue when trying to validate a relation representing a Doctrine's `ArrayCollection` (`toMany`). Fix the denormalization using the property getter. Return an `array` instead of an `ArrayCollection` with `$collectionRelation->getValues()`. Then, define your validation on the getter instead of the property.

For example:

```xml
<getter property="cars">
    <constraint name="Valid"/>
</getter>
```

```php
<?php

namespace App\Entity;

use Doctrine\Common\Collections\ArrayCollection;
use Symfony\Component\Validator\Constraints as Assert;

final class Brand
{
    // ...

    public function __construct()
    {
        $this->cars = new ArrayCollection();
    }

    /**
     * @Assert\Valid
     */
    public function getCars()
    {
        return $this->cars->getValues();
    }
}
```

## Open Vocabulary Generated from Validation Metadata

API Platform automatically detects Symfony's built-in validators and generates schema.org IRI metadata accordingly. This allows for rich clients such as the Admin component to infer the field types for most basic use cases.

The following validation constraints are covered:

Constraints                                                                           | Vocabulary                        |
--------------------------------------------------------------------------------------|-----------------------------------|
[`Url`](https://symfony.com/doc/current/reference/constraints/Url.html)               | `http://schema.org/url`           |
[`Email`](https://symfony.com/doc/current/reference/constraints/Email.html)           | `http://schema.org/email`         |
[`Uuid`](https://symfony.com/doc/current/reference/constraints/Uuid.html)             | `http://schema.org/identifier`    |
[`CardScheme`](https://symfony.com/doc/current/reference/constraints/CardScheme.html) | `http://schema.org/identifier`    |
[`Bic`](https://symfony.com/doc/current/reference/constraints/Bic.html)               | `http://schema.org/identifier`    |
[`Iban`](https://symfony.com/doc/current/reference/constraints/Iban.html)             | `http://schema.org/identifier`    |
[`Date`](https://symfony.com/doc/current/reference/constraints/Date.html)             | `http://schema.org/Date`          |
[`DateTime`](https://symfony.com/doc/current/reference/constraints/DateTime.html)     | `http://schema.org/DateTime`      |
[`Time`](https://symfony.com/doc/current/reference/constraints/Time.html)             | `http://schema.org/Time`          |
[`Image`](https://symfony.com/doc/current/reference/constraints/Image.html)           | `http://schema.org/image`         |
[`File`](https://symfony.com/doc/current/reference/constraints/File.html)             | `http://schema.org/MediaObject`   |
[`Currency`](https://symfony.com/doc/current/reference/constraints/Currency.html)     | `http://schema.org/priceCurrency` |
[`Isbn`](https://symfony.com/doc/current/reference/constraints/Isbn.html)             | `http://schema.org/isbn`          |
[`Issn`](https://symfony.com/doc/current/reference/constraints/Issn.html)             | `http://schema.org/issn`          |

## Specification property restrictions

API Platform generates specification property restrictions based on Symfony’s built-in validator.

For example, from [`Regex`](https://symfony.com/doc/4.4/reference/constraints/Regex.html) constraint API
 Platform builds [`pattern`](https://swagger.io/docs/specification/data-models/data-types/#pattern) restriction.

For building custom property schema based on custom validation constraints you can create a custom class
for generating property scheme restriction.

To create property schema, you have to implement the [`PropertySchemaRestrictionMetadataInterface`](https://github.com/api-platform/core/blob/caca7f26b7f22a0abf84390463a1ea47c47d7757/src/Bridge/Symfony/Validator/Metadata/Property/Restriction/PropertySchemaRestrictionMetadataInterface.php).
This interface defines only 2 methods:

* `create`: to create property schema
* `supports`: to check whether the property and constraint is supported

Here is an implementation example:

```php
namespace App\PropertySchemaRestriction;

use ApiPlatform\Core\Metadata\Property\PropertyMetadata;
use Symfony\Component\Validator\Constraint;
use App\Validator\CustomConstraint;

final class CustomPropertySchemaRestriction implements PropertySchemaRestrictionMetadataInterface
{
    public function supports(Constraint $constraint, PropertyMetadata $propertyMetadata): bool
    {
        return $constraint instanceof CustomConstraint;
    }

    public function create(Constraint $constraint, PropertyMetadata $propertyMetadata): array 
    {
      // your logic to create property schema restriction based on constraint
      return $restriction;
    }
}
```

If you use a custom dependency injection configuration, you need to register the corresponding service and add the
`api_platform.metadata.property_schema_restriction` tag. The `priority` attribute can be used for service ordering.

```yaml
# api/config/services.yaml
services:
    # ...
    'App\PropertySchemaRestriction\CustomPropertySchemaRestriction': ~
        # Uncomment only if autoconfiguration is disabled
        #tags: [ 'api_platform.metadata.property_schema_restriction' ]

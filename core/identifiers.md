# Identifiers

Every item operation has an identifier in its URL. Although this identifier is usually a number, it can also be an `UUID`, a date, or the type of your choice.
To help with your development experience, we introduced an identifier normalization process.

## Custom Identifier Normalizer

> In the following chapter, we're assuming that `App\Uuid` is a project-owned class that manages a time-based UUID.

Let's say you have the following class, which is identified by a `UUID` type. In this example, `UUID` is not a simple string but an object with many attributes.

```php
<?php
namespace App\Entity;

use App\Uuid;

/**
 * @ApiResource
 */
final class Person {
    /**
     * @var Uuid
     * @ApiProperty(identifier=true)
     */
    public $code;
    
    // ...
}
```

You can also use the YAML configuration format:

```yaml
App\Entity\Person:
    properties:
        code:
            identifier: true
        # ...
```

Once registered as an `ApiResource`, having an existing person, it will be accessible through the following URL: `/people/110e8400-e29b-11d4-a716-446655440000`.
Note that the property identifying our resource is named `code`.

Let's create a `DataProvider` for the `Person` entity:

```php
<?php
namespace App\DataProvider;

use App\Entity\Person;
use App\Uuid;

final class PersonDataProvider implements ItemDataProviderInterface, RestrictedDataProviderInterface {

    /**
     * {@inheritdoc}
     */
    public function getItem(string $resourceClass, $identifiers, string $operationName = null, array $context = [])
    {
        // Our identifier is:
        // $identifiers['code']
        // although it's a string, it's not an instance of Uuid and we wanted to retrieve the timestamp of our time-based uuid:
        // $identifiers['code']->getTimestamp()
    }

    /**
     * {@inheritdoc}
     */
    public function supports(string $resourceClass, string $operationName = null, array $context = []): bool
    {
        return $resourceClass === Person::class;
    }
}
```

To cover this use case, we need to `denormalize` the identifier to an instance of our `App\Uuid` class. This case is covered by an identifier denormalizer:

```php
<?php
namespace App\Identifier;

use ApiPlatform\Core\Exception\InvalidIdentifierException;
use App\Uuid;
use Symfony\Component\Serializer\Normalizer\DenormalizerInterface;

final class UuidNormalizer implements DenormalizerInterface
{
    /**
     * {@inheritdoc}
     */
    public function denormalize($data, $class, $format = null, array $context = [])
    {
        try {
            return Uuid::fromString($data);
        } catch (InvalidUuidStringException $e) {
            throw new InvalidIdentifierException($e->getMessage());
        }
    }

    /**
     * {@inheritdoc}
     */
    public function supportsDenormalization($data, $type, $format = null)
    {
        return is_a($type, Uuid::class, true);
    }
}
```

Tag this service as an `api_platform.identifier.denormalizer`:

```xml
  <service id="App\Identifier\UuidNormalizer" class="App\Identifier\UuidNormalizer" public="false">
      <tag name="api_platform.identifier.denormalizer" />
  </service>
```

```yaml
services:
    App\Identifier\UuidNormalizer:
        tags:
            - { name: api_platform.identifier.denormalizer }
```

Your `PersonDataProvider` will now work as expected!


## Supported Identifiers

API Platform supports the following identifier types:

  - `scalar` (string, integer)
  - `\DateTime` (uses the symfony `DateTimeNormalizer` internally, see [DateTimeIdentifierNormalizer](https://github.com/api-platform/core/blob/master/src/Identifier/Normalizer/DateTimeIdentifierDenormalizer.php))
  - `\Ramsey\Uuid\Uuid` (see [UuidNormalizer](https://github.com/api-platform/core/blob/master/src/Bridge/RamseyUuid/Identifier/Normalizer/UuidNormalizer.php))

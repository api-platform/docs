# Identifiers

Every item operation has an identifier in its URL. Although this identifier is usually a number, it can also be an `UUID`, a date, or the type of your choice.
To help with your development experience, we introduced an identifier normalization process.

## Custom Identifier Normalizer

> In the following chapter, we're assuming that `App\Uuid` is a project-owned class that manages a time-based UUID.

Let's say you have the following class, which is identified by a `UUID` type. In this example, `UUID` is not a simple string but an object with many attributes.

<code-selector>

```php
<?php
// api/src/ApiResource/Person.php with Symfony or app/ApiResource/Person.php with Laravel
namespace App\ApiResource;

use ApiPlatform\Metadata\ApiProperty;
use ApiPlatform\Metadata\ApiResource;
use App\State\PersonProvider;
use App\Uuid;

#[ApiResource(provider: PersonProvider::class)]
final class Person
{
    #[ApiProperty(identifier: true)]
    public Uuid $code;

    // ...
}
```

```yaml
# api/config/api_platform/resources/Person.yaml
# The YAML syntax is only supported for Symfony
properties:
  App\ApiResource\Person:
    code:
      identifier: true
resource:
  App\ApiResource\Person:
    provider: App\State\PersonProvider
```

```xml
<!-- The XML syntax is only supported for Symfony -->
<properties xmlns="https://api-platform.com/schema/metadata/properties-3.0">
    <property resource="App\ApiResource\Person" name="code" identifier="true"/>
</properties>
<resources xmlns="https://api-platform.com/schema/metadata/resources-3.0">
    <resource class="App\ApiResource\Person" provider="App\State\PersonProvider" />
</resources>
```

</code-selector>

Once registered as an `ApiResource`, having an existing person, it will be accessible through the following URL:
`/people/110e8400-e29b-11d4-a716-446655440000`. Note that the property identifying our resource is named `code`.

Let's create a `Provider` for the `Person` resource:

```php
<?php
// api/src/State/PersonProvider.php with Symfony or app/State/PersonProvider.php with Laravel

namespace App\State;

use App\ApiResource\Person;
use ApiPlatform\Metadata\Operation;
use ApiPlatform\State\ProviderInterface;
use App\Uuid;

/**
 * @implements ProviderInterface<Person>
 */
final class PersonProvider implements ProviderInterface
{
    public function provide(Operation $operation, array $uriVariables = [], array $context = []): Person
    {
        // Our identifier is:
        // $uriVariables['code']
        // although it's a string, it's not an instance of Uuid and we wanted to retrieve the timestamp of our time-based uuid:
        // $uriVariable['code']->getTimestamp()
    }
}
```

To cover this use case, we need to `transform` the identifier to an instance of our `App\Uuid` class.
This case is covered by an URI variable transformer:

```php
<?php
// api/src/Identifier/UuidUriVariableTransformer.php with Symfony or app/Identifier/UuidUriVariableTransformer.php with Laravel 
namespace App\Identifier;

use ApiPlatform\Api\UriVariableTransformerInterface;
use ApiPlatform\Exception\InvalidUriVariableException;
use App\Uuid;

final class UuidUriVariableTransformer implements UriVariableTransformerInterface
{
    /**
     * Transforms a uri variable value.
     *
     * @param mixed $value   The uri variable value to transform
     * @param array $types   The guessed type behind the uri variable
     * @param array $context Options available to the transformer
     *
     * @throws InvalidUriVariableException Occurs when the uriVariable could not be transformed
     */
     public function transform($value, array $types, array $context = []): Uuid
     {
        try {
            return Uuid::fromString($value);
        } catch (InvalidUuidStringException $e) {
            throw new InvalidUriVariableException($e->getMessage());
        }
     }

    /**
     * Checks whether the given uri variable is supported for transformation by this transformer.
     *
     * @param mixed $value   The uri variable value to transform
     * @param array $types   The types to which the data should be transformed
     * @param array $context Options available to the transformer
     */
    public function supportsTransformation($value, array $types, array $context = []): bool
    {
        foreach ($types as $type) {
            if (is_a($type, Uuid::class, true)) {
                return true;
            }
        }

        return false;
    }
}
```

Tag this service as an `api_platform.uri_variables.transformer` using one of the configurations below:

### Tag the Service using Symfony

<code-selector>

```yaml
# api/config/services.yaml
# The YAML syntax is only supported for Symfony

services:
  App\Identifier\UuidUriVariableTransformer:
    tags:
      - { name: api_platform.uri_variables.transformer }
```

```xml
<!-- The XML syntax is only supported for Symfony -->

  <service id="App\Identifier\UuidUriVariableTransformer" class="App\Identifier\UuidUriVariableTransformer" public="false">
      <tag name="api_platform.uri_variables.transformer" />
  </service>
```

</code-selector>

You can also use `#[AutoconfigureTag('api_platform.uri_variables.transformer')]` to tag the service.

```php
<?php

namespace App\Identifier;

use ApiPlatform\Api\UriVariableTransformerInterface;
use ApiPlatform\Exception\InvalidUriVariableException;
use App\Uuid;
use Symfony\Component\DependencyInjection\Attribute\AutoconfigureTag;
use Ramsey\Uuid\Exception\InvalidUuidStringException;

#[AutoconfigureTag('api_platform.uri_variables.transformer')]
final class UuidUriVariableTransformer implements UriVariableTransformerInterface
{
    /**
     * Transforms a URI variable value.
     *
     * @param mixed $value   The URI variable value to transform
     * @param array $types   The guessed type behind the URI variable
     * @param array $context Options available to the transformer
     *
     * @throws InvalidUriVariableException Occurs when the URI variable could not be transformed
     */
    public function transform($value, array $types, array $context = []): Uuid
    {
        try {
            return Uuid::fromString($value);
        } catch (InvalidUuidStringException $e) {
            throw new InvalidUriVariableException($e->getMessage());
        }
    }

    /**
     * Checks whether the given URI variable is supported for transformation by this transformer.
     *
     * @param mixed $value   The URI variable value to transform
     * @param array $types   The types to which the data should be transformed
     * @param array $context Options available to the transformer
     */
    public function supportsTransformation($value, array $types, array $context = []): bool
    {
        foreach ($types as $type) {
            if (is_a($type, Uuid::class, true)) {
                return true;
            }
        }

        return false;
    }
}
```

Your `PersonProvider` will now work as expected!

### Tag the Service using Laravel

```php
<?php

namespace App\Providers;

use App\Identifier\UuidUriVariableTransformer;
use ApiPlatform\Metadata\UriVariableTransformerInterface;
use Illuminate\Support\ServiceProvider;

class AppServiceProvider extends ServiceProvider
{
    public function register(): void
    {
        $this->app->tag([UuidUriVariableTransformer::class], UriVariableTransformerInterface::class);
    }
}
```

Your `PersonProvider` will now work as expected!

## Changing Identifier in a Doctrine Entity

If your resource is also a Doctrine entity and you want to use another identifier other than the Doctrine one, you have to unmark it:

```php
<?php
// api/src/Entity/Person.php

namespace App\Entity;

use ApiPlatform\Metadata\ApiProperty;
use ApiPlatform\Metadata\ApiResource;
use App\Uuid;
use Doctrine\ORM\Mapping as ORM;

#[ORM\Entity]
#[ApiResource]
final class Person
{
    #[ORM\Id, ORM\Column, ORM\GeneratedValue]
    #[ApiProperty(identifier: false)]
    private ?int $id = null;

    #[ORM\Column(type: 'uuid', unique: true)]
    #[ApiProperty(identifier: true)]
    public Uuid $code;

    // ...
}
```

## Supported Identifiers

API Platform supports the following identifier types:

- `scalar` (string, integer)
- `\DateTime` (uses the symfony `DateTimeNormalizer` internally, see [DateTimeIdentifierNormalizer](https://github.com/api-platform/core/blob/main/src/Api/UriVariableTransformer/DateTimeUriVariableTransformer.php))
- `\Ramsey\Uuid\Uuid` (see [UuidNormalizer](https://github.com/api-platform/core/blob/main/src/RamseyUuid/UriVariableTransformer/UuidUriVariableTransformer.php))
- `\Symfony\Component\Uid\Ulid` (see [UlidNormalizer](https://github.com/api-platform/core/blob/main/src/Symfony/UriVariableTransformer/UlidUriVariableTransformer.php))
- `\Symfony\Component\Uid\Uuid` (see [UuidNormalizer](https://github.com/api-platform/core/blob/main/src/Symfony/UriVariableTransformer/UuidUriVariableTransformer.php))
- `\Stringable` (essential when using composite identifiers from related resource classes)

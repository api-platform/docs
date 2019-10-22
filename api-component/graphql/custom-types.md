# Custom Types

You might need to add your own types to your GraphQL application.

Create your type class by implementing the interface `ApiPlatform\Core\GraphQl\Type\Definition\TypeInterface`.

You should extend the `GraphQL\Type\Definition\ScalarType` class too to take advantage of its useful methods.

For instance, to create a custom `DateType`:

```php
<?php

namespace App\Type\Definition;

use ApiPlatform\Core\GraphQl\Type\Definition\TypeInterface;
use GraphQL\Error\Error;
use GraphQL\Language\AST\StringValueNode;
use GraphQL\Type\Definition\ScalarType;
use GraphQL\Utils\Utils;

final class DateTimeType extends ScalarType implements TypeInterface
{
    public function __construct()
    {
        $this->name = 'DateTime';
        $this->description = 'The `DateTime` scalar type represents time data.';

        parent::__construct();
    }

    public function getName(): string
    {
        return $this->name;
    }

    /**
     * {@inheritdoc}
     */
    public function serialize($value)
    {
        // Already serialized.
        if (\is_string($value)) {
            return (new \DateTime($value))->format('Y-m-d');
        }

        if (!($value instanceof \DateTime)) {
            throw new Error(sprintf('Value must be an instance of DateTime to be represented by DateTime: %s', Utils::printSafe($value)));
        }

        return $value->format(\DateTime::ATOM);
    }

    /**
     * {@inheritdoc}
     */
    public function parseValue($value)
    {
        if (!\is_string($value)) {
            throw new Error(sprintf('DateTime cannot represent non string value: %s', Utils::printSafeJson($value)));
        }

        if (false === \DateTime::createFromFormat(\DateTime::ATOM, $value)) {
            throw new Error(sprintf('DateTime cannot represent non date value: %s', Utils::printSafeJson($value)));
        }

        // Will be denormalized into a \DateTime.
        return $value;
    }

    /**
     * {@inheritdoc}
     */
    public function parseLiteral($valueNode, ?array $variables = null)
    {
        if ($valueNode instanceof StringValueNode && false !== \DateTime::createFromFormat(\DateTime::ATOM, $valueNode->value)) {
            return $valueNode->value;
        }

        // Intentionally without message, as all information already in wrapped Exception
        throw new \Exception();
    }
}
```

You can also check the documentation of [graphql-php](https://webonyx.github.io/graphql-php/type-system/scalar-types/#writing-custom-scalar-types).

The big difference in API Platform is that the value is already serialized when it's received in your type class.
Similarly, you would not want to denormalize your parsed value since it will be done by API Platform later.

If you use autoconfiguration (the default Symfony configuration) in your application, then you are done!

Else, you need to tag your type class like this:

```yaml
# api/config/services.yaml
services:
    # ...
    App\Type\Definition\DateTimeType:
        tags:
            - { name: api_platform.graphql.type }
```

Your custom type is now registered and is available in the `TypesContainer`.

To use it please [modify the extracted types](#modify-the-extracted-types) or use it directly in [custom queries](queries.md) or [custom mutations](mutations.md).

## Modify the Extracted Types

The GraphQL schema and its types are extracted from your resources.
In some cases, you would want to modify the extracted types for instance to use your custom ones.

To do so, you need to decorate the `api_platform.graphql.type_converter` service:

```yaml
# api/config/services.yaml
services:
    # ...
    'App\Type\TypeConverter':
        decorates: api_platform.graphql.type_converter
```

Your class needs to look like this:

```php
<?php

namespace App\Type;

use ApiPlatform\Core\GraphQl\Type\TypeConverterInterface;
use App\Model\Book;
use Symfony\Component\PropertyInfo\Type;

final class TypeConverter implements TypeConverterInterface
{
    private $defaultTypeConverter;

    public function __construct(TypeConverterInterface $defaultTypeConverter)
    {
        $this->defaultTypeConverter = $defaultTypeConverter;
    }

    /**
     * {@inheritdoc}
     */
    public function convertType(Type $type, bool $input, ?string $queryName, ?string $mutationName, string $resourceClass, string $rootResource, ?string $property, int $depth)
    {
        if ('publicationDate' === $property
            && Book::class === $resourceClass
        ) {
            return 'DateTime';
        }

        return $this->defaultTypeConverter->convertType($type, $input, $queryName, $mutationName, $resourceClass, $rootResource, $property, $depth);
    }
}
```

In this case, the `publicationDate` property of the `Book` class will have a custom `DateTime` type.

You can even apply this logic for a kind of property. Replace the previous condition with something like this:

```php
if (Type::BUILTIN_TYPE_OBJECT === $type->getBuiltinType()
    && is_a($type->getClassName(), \DateTimeInterface::class, true)
) {
    return 'DateTime';
}
```

All `DateTimeInterface` properties will have the `DateTime` type in this example.

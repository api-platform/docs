# Use snake_case instead of camelCase

The Serializer Component provides a handy way to map PHP field names to serialized names. See the related [Symfony documentation](https://symfony.com/doc/master/components/serializer.html#converting-property-names-when-serializing-and-deserializing).

To use this feature, declare a new name converter service. For example, you can convert `CamelCase` to
`snake_case` with the following configuration:

```yaml
# api/config/services.yaml
services:
    'Symfony\Component\Serializer\NameConverter\CamelCaseToSnakeCaseNameConverter': ~
```

```yaml
# api/config/packages/api_platform.yaml
api_platform:
    name_converter: 'Symfony\Component\Serializer\NameConverter\CamelCaseToSnakeCaseNameConverter'
```

If Symfony's `MetadataAwareNameConverter` is available it'll be used by default. If you specify one in ApiPlatform configuration, it'll be used. Note that you can use decoration to benefit from this name converter in your own implementation.

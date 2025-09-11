# Parameters and Filters

API Platform provides a generic and powerful system to apply filters, sort criteria, and handle other request parameters. This system is primarily managed through **Parameter attributes** (`#[QueryParameter]` and `#[HeaderParameter]`), which allow for detailed and explicit configuration of how an API consumer can interact with a resource.

These parameters can be linked to **Filters**, which are classes that contain the logic for applying criteria to your persistence backend (like Doctrine ORM or MongoDB ODM).

You can declare parameters on a resource class to apply them to all operations, or on a specific operation for more granular control. When parameters are enabled, they automatically appear in the Hydra, [OpenAPI](openapi.md) and [GraphQL](graphql.md) documentations.

<p align="center" class="symfonycasts"><a href="https://symfonycasts.com/screencast/api-platform/filters?cid=apip"><img src="../symfony/images/symfonycasts-player.png" alt="Filtering and Searching screencast"><br>Watch the Filtering & Searching screencast</a></p>

For documentation on the specific filter implementations available for your persistence layer, please refer to the following pages:

* [Doctrine Filters](../core/doctrine-filters.md)
* [Elasticsearch Filters](../core/elasticsearch-filters.md)

## Declaring Parameters

The recommended way to define parameters is by using Parameter attributes directly on a resource class or on an operation. API Platform provides two main types of Parameter attributes based on their location (matching the OpenAPI `in` configuration):

* `ApiPlatform\Metadata\QueryParameter`: For URL query parameters (e.g., `?name=value`).
* `ApiPlatform\Metadata\HeaderParameter`: For HTTP headers (e.g., `Custom-Header: value`).

You can declare a parameter on the resource class to make it available for all its operations:

```php
<?php
// api/src/Resource/Book.php
namespace App\Resource;

use ApiPlatform\Metadata\ApiResource;
use ApiPlatform\Metadata\QueryParameter;

#[ApiResource]
#[QueryParameter(key: 'author')]
class Book
{
    // ...
}
```

Or you can declare it on a specific operation for more targeted use cases:

```php
<?php
// api/src/Resource/Friend.php
namespace App\Resource;

use ApiPlatform\Metadata\ApiResource;
use ApiPlatform\Metadata\Get;
use ApiPlatform\Metadata\HeaderParameter;
use ApiPlatform\Metadata\QueryParameter;

#[ApiResource(
    operations: [
        new GetCollection(
            parameters: [
                'name' => new QueryParameter(description: 'Filter our friends by name'),
                'Request-ID' => new HeaderParameter(description: 'A unique request identifier') // keys are case insensitive
            ]
        )
    ]
)]
class Friend
{
    // ...
}
```

### Filtering a Single Property

Most of the time, a parameter maps directly to a property on your resource. For example, a `?name=Frodo` query parameter would filter for resources where the `name` property is "Frodo". This behavior is often handled by built-in or custom filters that you link to the parameter.

For Hydra, you can map a query parameter to `hydra:freetextQuery` to indicate a general-purpose search query.

```php
<?php
// api/src/Resource/Issue.php
use ApiPlatform\Metadata\ApiResource;
use ApiPlatform\Metadata\GetCollection;
use ApiPlatform\Metadata\QueryParameter;

#[ApiResource(operations: [
    new GetCollection(
        parameters: [
            'q' => new QueryParameter(property: 'hydra:freetextQuery', required: true)
        ]
    )
])]
class Issue {}
```

This will generate the following Hydra `IriTemplateMapping`:
```json
{
  "@context": "http://www.w3.org/ns/hydra/context.jsonld",
  "@type": "IriTemplate",
  "template": "http://api.example.com/issues{?q}",
  "variableRepresentation": "BasicRepresentation",
  "mapping": [
    {
      "@type": "IriTemplateMapping",
      "variable": "q",
      "property": "hydra:freetextQuery",
      "required": true
    }
  ]
}
```

### Filtering Multiple Properties with `:property`

Sometimes you need a generic filter that can operate on multiple properties. You can achieve this by using the `:property` placeholder in the parameter's `key`.

```php
<?php
// api/src/Resource/Book.php
use ApiPlatform\Doctrine\Orm\Filter\SearchFilter;
use ApiPlatform\Metadata\ApiResource;
use ApiPlatform\Metadata\GetCollection;
use ApiPlatform\Metadata\QueryParameter;

#[ApiResource(operations: [
    new GetCollection(
        parameters: [
            'search[:property]' => new QueryParameter(
                filter: 'api_platform.doctrine.orm.search_filter.instance'
            )
        ]
    )
])]
class Book 
{
    // ...
}
```

This configuration creates a dynamic parameter. API clients can now filter on any of the properties configured in the `SearchFilter` (in this case, `title` and `description`) by using a URL like `/books?search[title]=Ring` or `/books?search[description]=journey`.

When using the `:property` placeholder, API Platform automatically populates the parameter's `extraProperties` with a `_properties` array containing all the available properties for the filter. Your filter can access this information:

```php
public function apply(QueryBuilder $queryBuilder, QueryNameGeneratorInterface $queryNameGenerator, string $resourceClass, ?Operation $operation = null, array $context = []): void
{
    $parameter = $context['parameter'] ?? null;
    $properties = $parameter?->getExtraProperties()['_properties'] ?? [];
    
    // $properties contains: ['title' => 'title', 'description' => 'description']
    // This allows your filter to know which properties are available for filtering
}
```

Note that we're using `api_platform.doctrine.orm.search_filter.instance` (exists also for ODM). Indeed this is a special instance of the search filter where `properties` can be changed during runtime. This is considered as "legacy filter" below, in API Platform 4.0 we'll recommend to create a custom filter or to use the `PartialSearchFilter`.

### Restricting Properties with `:property` Placeholders

There are two different approaches to property restriction depending on your filter design:

#### 1. Legacy Filters (SearchFilter, etc.) - Not Recommended

> [!WARNING]
> Filters that extend `AbstractFilter` with pre-configured properties are considered legacy. They don't support property restriction via parameters and may be deprecated in future versions. Consider using per-parameter filters instead for better flexibility and performance.

For existing filters that extend `AbstractFilter` and have pre-configured properties, the parameter's `properties` does **not** restrict the filter's behavior. These filters use their own internal property configuration:

```php
<?php
// This does NOT restrict SearchFilter - it processes all its configured properties
'search[:property]' => new QueryParameter(
    properties: ['title', 'author'], // Only affects _properties, doesn't restrict filter
    filter: new SearchFilter(properties: ['title' => 'partial', 'description' => 'partial'])
)

// To restrict legacy filters, configure them with only the desired properties:
'search[:property]' => new QueryParameter(
    filter: new SearchFilter(properties: ['title' => 'partial', 'author' => 'exact'])
)
```

#### 2. Per-Parameter Filters (Recommended)

> [!NOTE]
> Per-parameter filters are the modern approach. They provide better performance (only process requested properties), cleaner code, and full support for parameter-based property restriction.

Modern filters that work on a per-parameter basis can be effectively restricted using the parameter's `properties`:

```php
<?php
// src/Filter/PartialSearchFilter.php
use ApiPlatform\Doctrine\Orm\Filter\FilterInterface;
use ApiPlatform\Doctrine\Orm\Util\QueryNameGeneratorInterface;
use ApiPlatform\Metadata\Operation;
use Doctrine\ORM\QueryBuilder;

final class PartialSearchFilter implements FilterInterface
{
    public function apply(QueryBuilder $queryBuilder, QueryNameGeneratorInterface $queryNameGenerator, string $resourceClass, ?Operation $operation = null, array $context = []): void
    {
        $parameter = $context['parameter'];
        $value = $parameter->getValue();
        
        // Get the property for this specific parameter
        $property = $parameter->getProperty();
        $alias = $queryBuilder->getRootAliases()[0];
        $field = $alias.'.'.$property;
        
        $parameterName = $queryNameGenerator->generateParameterName($property);
        
        $queryBuilder
            ->andWhere($queryBuilder->expr()->like('LOWER('.$field.')', ':'.$parameterName))
            ->setParameter($parameterName, '%'.strtolower($value).'%');
    }
}
```

```php
<?php
// api/src/Resource/Book.php
#[ApiResource(operations: [
    new GetCollection(
        parameters: [
            // This WILL restrict to only title and author properties
            'search[:property]' => new QueryParameter(
                properties: ['title', 'author'], // Only these properties get parameters created
                filter: new PartialSearchFilter()
            )
        ]
    )
])]
class Book {
    // ...
}
```

**How it works:**
1. API Platform creates individual parameters: `search[title]` and `search[author]` only
2. URLs like `/books?search[description]=foo` are ignored (no parameter exists)
3. Each parameter calls the filter with its specific property via `$parameter->getProperty()`
4. The filter processes only that one property

This approach is recommended for new filters as it's more flexible and allows true property restriction via the parameter configuration.

Note that invalid values are usually ignored by our filters, use [validation](#parameter-validation) to trigger errors for wrong parameter values.

## OpenAPI and JSON Schema

You have full control over how your parameters are documented in OpenAPI.

### Customizing the OpenAPI Parameter

You can pass a fully configured `ApiPlatform\OpenApi\Model\Parameter` object to the `openApi` property of your parameter attribute. This gives you total control over the generated documentation.

```php
<?php
// api/src/Resource/User.php
use ApiPlatform\Metadata\ApiResource;
use ApiPlatform\Metadata\GetCollection;
use ApiPlatform\Metadata\QueryParameter;
use ApiPlatform\OpenApi\Model\Parameter as OpenApiParameter;

#[ApiResource(operations: [
    new GetCollection(
        uriTemplate: '/users/validate_openapi',
        parameters: [
            'enum' => new QueryParameter(
                schema: ['enum' => ['a', 'b'], 'uniqueItems' => true],
                castToArray: true,
                openApi: new OpenApiParameter(name: 'enum', in: 'query', style: 'deepObject')
            )
        ]
    )
])]
class User {}
```

### Using JSON Schema and Type Casting

The `schema` property allows you to define validation rules using JSON Schema keywords. This is useful for simple validation like ranges, patterns, or enumerations.

When you define a `schema`, API Platform can often infer the native PHP type of the parameter. For instance, `['type' => 'boolean']` implies a boolean. If you want to ensure the incoming string value (e.g., "true", "0") is cast to its actual native type before validation and filtering, set `castToNativeType` to `true`.

```php
<?php
// api/src/Resource/Setting.php
use ApiPlatform\Metadata\ApiResource;
use ApiPlatform\Metadata\GetCollection;
use ApiPlatform\Metadata\QueryParameter;

#[ApiResource(operations: [
    new GetCollection(
        uriTemplate: '/settings',
        parameters: [
            'isEnabled' => new QueryParameter(
                schema: ['type' => 'boolean'],
                castToNativeType: true
            )
        ]
    )
])]
class Setting {}
```

If you need a custom validation function use the `castFn` property of the `Parameter` class.

## Parameter Validation

You can enforce validation rules on your parameters using the `required` property or by attaching Symfony Validator constraints.

```php
<?php
// api/src/Resource/User.php
use ApiPlatform\Metadata\ApiResource;
use ApiPlatform\Metadata\GetCollection;
use ApiPlatform\Metadata\HeaderParameter;
use ApiPlatform\Metadata\QueryParameter;
use Symfony\Component\Validator\Constraints as Assert;

#[ApiResource(operations: [
    new GetCollection(
        uriTemplate: '/users/validate',
        parameters: [
            'country' => new QueryParameter(
                description: 'Filter by country code.',
                constraints: [new Assert\Country()]
            ),
            'X-Request-ID' => new HeaderParameter(
                description: 'A unique request identifier.',
                required: true,
                constraints: [new Assert\Uuid()]
            )
        ]
    )
])]
class User {}
```

Note that when `castToNativeType` is enabled, API Platform infers type validation from the JSON Schema.

The `ApiPlatform\Validator\Util\ParameterValidationConstraints` trait can be used to automatically infer validation constraints from the JSON Schema and OpenAPI definitions of a parameter.

Here is the list of validation constraints that are automatically inferred from the JSON Schema and OpenAPI definitions of a parameter.

### From OpenAPI Definition

* **`allowEmptyValue`**: If set to `false`, a `Symfony\Component\Validator\Constraints\NotBlank` constraint is added.

### From JSON Schema (`schema` property)

* **`minimum`** / **`maximum`**:
  * If both are set, a `Symfony\Component\Validator\Constraints\Range` constraint is added.
  * If only `minimum` is set, a `Symfony\Component\Validator\Constraints\GreaterThanOrEqual` constraint is added.
  * If only `maximum` is set, a `Symfony\Component\Validator\Constraints\LessThanOrEqual` constraint is added.
* **`exclusiveMinimum`** / **`exclusiveMaximum`**:
  * If `exclusiveMinimum` is used, it becomes a `Symfony\Component\Validator\Constraints\GreaterThan` constraint.
  * If `exclusiveMaximum` is used, it becomes a `Symfony\Component\Validator\Constraints\LessThan` constraint.
* **`pattern`**: Becomes a `Symfony\Component\Validator\Constraints\Regex` constraint.
* **`minLength`** / **`maxLength`**: Becomes a `Symfony\Component\Validator\Constraints\Length` constraint.
* **`multipleOf`**: Becomes a `Symfony\Component\Validator\Constraints\DivisibleBy` constraint.
* **`enum`**: Becomes a `Symfony\Component\Validator\Constraints\Choice` constraint with the specified values.
* **`minItems`** / **`maxItems`**: Becomes a `Symfony\Component\Validator\Constraints\Count` constraint (for arrays).
* **`uniqueItems`**: If `true`, becomes a `Symfony\Component\Validator\Constraints\Unique` constraint (for arrays).
* **`type`**:
  * If set to `'array'`, a `Symfony\Component\Validator\Constraints\Type('array')` constraint is added.
  * If `castToNativeType` is also `true`, the schema `type` will add a `Symfony\Component\Validator\Constraints\Type` constraint for `'boolean'`, `'integer'`, and `'number'` (as `float`).

### From the Parameter's `required` Property

* **`required`**: If set to `true`, a `Symfony\Component\Validator\Constraints\NotNull` constraint is added.

### Strict Parameter Validation

By default, API Platform allows clients to send extra query parameters that are not defined in the operation's `parameters`. To enforce a stricter contract, you can set `strictQueryParameterValidation` to `true` on an operation. If an unsupported parameter is sent, API Platform will return a 400 Bad Request error.

```php
<?php
// api/src/Resource/StrictParameters.php
use ApiPlatform\Metadata\ApiResource;
use ApiPlatform\Metadata\Get;
use ApiPlatform\Metadata\QueryParameter;

#[ApiResource(operations: [
    new Get(
        uriTemplate: 'strict_query_parameters',
        strictQueryParameterValidation: true,
        parameters: [
            'foo' => new QueryParameter(),
        ]
    )
])]
class StrictParameters {}
```

With this configuration, a request to `/strict_query_parameters?bar=test` will fail with a 400 error because `bar` is not a supported parameter.

## Parameter Providers

Parameter Providers are powerful services that can inspect, transform, or provide values for parameters. They can even modify the current `Operation` metadata on the fly. A provider is a class that implements `ApiPlatform\State\ParameterProviderInterface`.

### `IriConverterParameterProvider`

This built-in provider takes an IRI string (e.g., `/users/1`) and converts it into the corresponding Doctrine entity object. It supports both single IRIs and arrays of IRIs.

```php
<?php
// api/src/Resource/WithParameter.php
use ApiPlatform\Metadata\ApiResource;
use ApiPlatform\Metadata\Get;
use ApiPlatform\Metadata\Operation;
use ApiPlatform\Metadata\QueryParameter;
use ApiPlatform\State\ParameterProvider\IriConverterParameterProvider;

#[ApiResource(operations: [
    new Get(
        uriTemplate: '/with_parameters_iris',
        parameters: [
            'dummy' => new QueryParameter(provider: IriConverterParameterProvider::class),
            'related' => new QueryParameter(
                provider: IriConverterParameterProvider::class,
                extraProperties: ['fetch_data' => true] // Forces fetching the entity data
            ),
        ],
        provider: [self::class, 'provideDummyFromParameter'],
    )
])]
class WithParameter 
{
    public static function provideDummyFromParameter(Operation $operation, array $uriVariables = [], array $context = []): object|array
    {
        // The value has been transformed from an IRI to an entity by the provider.
        $dummy = $operation->getParameters()->get('dummy')->getValue();
        
        // If multiple IRIs were provided as an array, this will be an array of entities
        $related = $operation->getParameters()->get('related')->getValue();
        
        return $dummy;
    }
}
```

#### Configuration Options

The `IriConverterParameterProvider` supports the following options in `extraProperties`:

- **`fetch_data`**: Boolean (default: `false`) - When `true`, forces the IRI converter to fetch the actual entity data instead of just creating a reference.

### `ReadLinkParameterProvider`

This provider fetches a linked resource from a given identifier. This is useful when you need to load a related entity to use later, for example in your own state provider.
When you have an API resource with a custom `uriTemplate` that includes parameters, the `ReadLinkParameterProvider` can automatically resolve the linked resource using the operation's URI template. This is particularly useful for nested resources or when you need to load a parent resource based on URI variables.

```php
<?php
// api/src/Resource/WithParameter.php
use ApiPlatform\Metadata\ApiResource;
use ApiPlatform\Metadata\Get;
use ApiPlatform\Metadata\Link;
use ApiPlatform\Metadata\Operation;
use ApiPlatform\Metadata\QueryParameter;
use ApiPlatform\State\ParameterProvider\ReadLinkParameterProvider;
use App\Entity\Dummy;

#[Get(
    uriTemplate: 'with_parameters/{id}{._format}',
    uriVariables: [
        'id' => new Link(schema: ['type' => 'string', 'format' => 'uuid'], property: 'id'),
    ],
    parameters: [
        'dummy' => new QueryParameter(
            provider: ReadLinkParameterProvider::class, 
            extraProperties: [
                'resource_class' => Dummy::class,
                'uri_template' => '/dummies/{id}' // Optional: specify the template for the linked resource
            ]
        )
    ],
    provider: [self::class, 'provideDummyFromParameter'],
)]
class WithParameter 
{
    public static function provideDummyFromParameter(Operation $operation, array $uriVariables = [], array $context = []): object|array
    {
        // The dummy parameter has been resolved to the actual Dummy entity
        // based on the parameter value and the specified uri_template
        return $operation->getParameters()->get('dummy')->getValue();
    }
}
```

The provider will:
- Take the parameter value (e.g., a UUID or identifier)
- Use the `resource_class` to determine which resource to load
- Optionally use the `uri_template` from `extraProperties` to construct the proper operation for loading the resource
- Return the loaded entity, making it available in your state provider

#### ReadLinkParameterProvider Configuration Options

You can control the behavior of `ReadLinkParameterProvider` with these `extraProperties`:

- **`resource_class`**: The class of the resource to load
- **`uri_template`**: Optional URI template for the linked resource operation
- **`uri_variable`**: Name of the URI variable to use when building URI variables array
- **`throw_not_found`**: Boolean (default: `true`) - Whether to throw `NotFoundHttpException` when resource is not found

```php
'dummy' => new QueryParameter(
    provider: ReadLinkParameterProvider::class, 
    extraProperties: [
        'resource_class' => Dummy::class,
        'throw_not_found' => false, // Won't throw NotFoundHttpException if resource is missing
        'uri_variable' => 'customId' // Use 'customId' as the URI variable name
    ]
)
```

### Array Support

Both `IriConverterParameterProvider` and `ReadLinkParameterProvider` support processing arrays of values. When you pass an array of identifiers or IRIs, they will return an array of resolved entities:

```php
// For IRI converter: ?related[]=/dummies/1&related[]=/dummies/2
// For ReadLink provider: ?dummies[]=uuid1&dummies[]=uuid2
'items' => new QueryParameter(
    provider: ReadLinkParameterProvider::class,
    extraProperties: ['resource_class' => Dummy::class]
)
```

### Creating a Custom Parameter Provider

You can create your own providers to implement any custom logic. A provider must implement `ParameterProviderInterface`. The `provide` method can modify the parameter's value or even return a modified `Operation` to alter the request handling flow.

For instance, a provider could add serialization groups to the normalization context based on a query parameter:

```php
<?php
// src/State/DynamicGroupProvider.php
namespace App\State;

use ApiPlatform\Metadata\Operation;
use ApiPlatform\Metadata\Parameter;
use ApiPlatform\State\ParameterProviderInterface;
use Symfony\Component\Serializer\Normalizer\AbstractNormalizer;

final class DynamicGroupProvider implements ParameterProviderInterface
{
    public function provide(Parameter $parameter, array $parameters = [], array $context = []): ?Operation
    {
        $operation = $context['operation'] ?? null;
        if (!$operation) {
            return null;
        }
        
        $value = $parameter->getValue();
        if ('extended' === $value) {
            $context = $operation->getNormalizationContext();
            $context[AbstractNormalizer::GROUPS][] = 'extended_read';
            return $operation->withNormalizationContext($context);
        }

        return $operation;
    }
}
```

### Changing how to parse Query / Header Parameters

We use our own algorithm to parse a request's query, if you want to do the parsing of `QUERY_STRING` yourself, set `_api_query_parameters` in the Request attributes (`$request->attributes->set('_api_query_parameters', [])`) yourself.
By default we use Symfony's `$request->headers->all()`, you can also set `_api_header_parameters` if you want to parse them yourself.

## Creating Custom Filters

For data-provider-specific filtering (e.g., Doctrine ORM), the recommended way to create a filter is to implement the corresponding `FilterInterface`.

For Doctrine ORM, your filter should implement `ApiPlatform\Doctrine\Orm\Filter\FilterInterface`:

```php
<?php
// src/Filter/RegexpFilter.php
namespace App\Filter;

use ApiPlatform\Doctrine\Orm\Filter\FilterInterface;
use ApiPlatform\Metadata\Operation;
use ApiPlatform\State\ParameterNotFound;
use Doctrine\ORM\QueryBuilder;
use Doctrine\ORM\Query\QueryNameGeneratorInterface;

final class RegexpFilter implements FilterInterface
{
    public function apply(QueryBuilder $queryBuilder, QueryNameGeneratorInterface $queryNameGenerator, string $resourceClass, ?Operation $operation = null, array $context = []): void
    {
        $parameter = $context['parameter'] ?? null;
        $value = $parameter?->getValue();

        // The parameter may not be present.
        // It's recommended to add validation (e.g., `required: true`) on the Parameter attribute
        // if the filter logic depends on the value.
        if ($value instanceof ParameterNotFound) {
            return;
        }

        $alias = $queryBuilder->getRootAliases()[0];
        $parameterName = $queryNameGenerator->generateParameterName('regexp_name');
        
        // Access the parameter's property or use the parameter key as fallback
        $property = $parameter->getProperty() ?? $parameter->getKey() ?? 'name';
        
        // You can also access filter context if the parameter provides it
        $filterContext = $parameter->getFilterContext() ?? null;

        $queryBuilder
            ->andWhere(sprintf('REGEXP(%s.%s, :%s) = 1', $alias, $property, $parameterName))
            ->setParameter($parameterName, $value);
    }

    // For BC, this function is not useful anymore when documentation occurs on the Parameter
    public function getDescription(): array {
        return [];
    }
}
```

You can then instantiate this filter directly in your `QueryParameter`:

```php
<?php
// api/src/Entity/User.php
use ApiPlatform\Metadata\ApiResource;
use ApiPlatform\Metadata\GetCollection;
use ApiPlatform\Metadata\QueryParameter;
use App\Filter\RegexpFilter;

#[ApiResource(operations: [
    new GetCollection(
        parameters: [
            'regexp_name' => new QueryParameter(filter: new RegexpFilter())
        ]
    )
])]
class User {}
```

### Advanced Use Case: Composing Filters

You can create complex filters by composing existing ones. This is useful when you want to apply multiple filtering logics based on a single parameter.

```php
<?php
// src/Filter/SearchTextAndDateFilter.php
namespace App\Filter;

use ApiPlatform\Doctrine\Orm\Filter\FilterInterface;
use ApiPlatform\Metadata\Operation;
use Doctrine\ORM\QueryBuilder;
use Doctrine\ORM\Query\QueryNameGeneratorInterface;
use Symfony\Component\DependencyInjection\Attribute\Autowire;

final class SearchTextAndDateFilter implements FilterInterface
{
    public function __construct(
        #[Autowire('@api_platform.doctrine.orm.search_filter.instance')] 
        public readonly FilterInterface $searchFilter, 
        #[Autowire('@api_platform.doctrine.orm.date_filter.instance')] 
        public readonly FilterInterface $dateFilter
    ) {}

    public function apply(QueryBuilder $queryBuilder, QueryNameGeneratorInterface $queryNameGenerator, string $resourceClass, ?Operation $operation = null, array $context = []): void
    {
        $value = $context['parameter']?->getValue();
        if ($value instanceof ParameterNotFound) {
            return;
        }

        // Create a new context for the sub-filters, passing the value.
        $subContext = ['filters' => ['searchOnTextAndDate' => $value]] + $context;

        $this->searchFilter->apply($queryBuilder, $queryNameGenerator, $resourceClass, $operation, $subContext);
        $this->dateFilter->apply($queryBuilder, $queryNameGenerator, $resourceClass, $operation, $subContext);
    }
}
```

To use this composite filter, register it as a service and reference it by its ID:

```yaml
# config/services.yaml
services:
    'app.filter_date_and_search':
        class: App\Filter\SearchTextAndDateFilter
        autowire: true
```php
<?php
// ...
#[ApiResource(operations: [
    new GetCollection(
        parameters: [
            'searchOnTextAndDate' => new QueryParameter(filter: 'app.filter_date_and_search')
        ]
    )
])]
class LogEntry {}
```

## Parameter Attribute Reference

| Property | Description |
|---|---|
| `key` | The name of the parameter (e.g., `name`, `order`). |
| `filter` | The filter service or instance that processes the parameter's value. |
| `provider` | A service that transforms the parameter's value before it's used. |
| `description` | A description for the API documentation. |
| `property` | The resource property this parameter is mapped to. |
| `required` | Whether the parameter is required. |
| `constraints` | Symfony Validator constraints to apply to the value. |
| `schema` | A JSON Schema for validation and documentation. |
| `castToArray` | Casts the parameter value to an array. Useful for query parameters like `foo[]=1&foo[]=2`. Defaults to `true`. |
| `castToNativeType` | Casts the parameter value to its native PHP type based on the `schema`. |
| `openApi` | Customize OpenAPI documentation or hide the parameter (`false`). |
| `hydra` | Hide the parameter from Hydra documentation (`false`). |
| `security` | A [Symfony expression](https://symfony.com/doc/current/security/expressions.html) to control access to the parameter. |

## Parameter Security

You can secure individual parameters using Symfony expression language. When a security expression evaluates to `false`, the parameter will be ignored and treated as if it wasn't provided.

```php
<?php
// api/src/Resource/SecureResource.php
use ApiPlatform\Metadata\ApiResource;
use ApiPlatform\Metadata\GetCollection;
use ApiPlatform\Metadata\HeaderParameter;
use ApiPlatform\Metadata\QueryParameter;

#[ApiResource(operations: [
    new GetCollection(
        uriTemplate: '/secure_resources',
        parameters: [
            'name' => new QueryParameter(
                security: 'is_granted("ROLE_ADMIN")'
            ),
            'auth' => new HeaderParameter(
                security: '"secured" == auth',
                description: 'Only accessible when auth header equals "secured"'
            ),
            'secret' => new QueryParameter(
                security: '"secured" == secret',
                description: 'Only accessible when secret parameter equals "secured"'
            )
        ]
    )
])]
class SecureResource
{
    // ...
}
```

In the security expressions, you have access to:
- Parameter values by their key name (e.g., `auth`, `secret`)
- Standard security functions like `is_granted()`
- The current user via `user`
- Request object via `request`


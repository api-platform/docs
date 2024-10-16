# Filters

API Platform provides a generic system to apply filters and sort criteria on collections.
Useful filters for Doctrine ORM, Eloquent ORM, MongoDB ODM and ElasticSearch are provided with the library.

You can also create custom filters that fit your specific needs.
You can also add filtering support to your custom [state providers](state-providers.md) by implementing interfaces provided
by the library.

By default, all filters are disabled. They must be enabled explicitly.

When a filter is enabled, it automatically appears in the [OpenAPI](openapi.md) and [GraphQL](graphql.md) documentations.
It is also automatically documented as a `search` property for JSON-LD responses.

<p align="center" class="symfonycasts"><a href="https://symfonycasts.com/screencast/api-platform/filters?cid=apip"><img src="../symfony/images/symfonycasts-player.png" alt="Filtering and Searching screencast"><br>Watch the Filtering & Searching screencast</a></p>

For the **specific filters documentation**, please refer to the following pages, depending on your needs:
- [Doctrine filters documentation](doctrine-filters.md)
- [Elasticsearch filters documentation](elasticsearch-filters)
- [Laravel filters documentation](../laravel/filters.md)

## Parameters

You can declare parameters on a Resource or an Operation through the `parameters` property.

```php
namespace App\ApiResource;

use ApiPlatform\Metadata\GetCollection;
use ApiPlatform\Metadata\QueryParameter;

// This parameter "page" works only on /books
#[GetCollection(uriTemplate: '/books', parameters: ['page' => new QueryParameter])]
// This parameter is available on every operations, key is mandatory
#[QueryParameter(key: 'q', property: 'freetextQuery')]
class Book {}
```

Note that `property` is used to document the Hydra view. You can also specify an [OpenAPI Parameter](https://api-platform.com/docs/references/OpenApi/Model/Parameter/) if needed.
A Parameter can be linked to a filter, there are two types of filters:

- metadata filters, most common are serializer filters (PropertyFilter and GroupFilter) that alter the normalization context
- query filters that alter the results of your database queries (Doctrine, Eloquent, Elasticsearch etc.)

### Alter the Operation via a parameter

A parameter can alter the current Operation context, to do so use a `ApiPlatform\State\ParameterProviderInterface`:

```php
class GroupsParameterProvider implements ParameterProviderInterface {
    public function provide(Parameter $parameter, array $uriVariables = [], array $context = []): HttpOperation
    {
        $request = $context['request'];
        return $context['operation']->withNormalizationContext(['groups' => $request->query->all('groups')]);
    }
}
```

Then plug this provider on your parameter:

```php
namespace App\ApiResource;

use ApiPlatform\Metadata\HeaderParameter;

#[Get(parameters: ['groups' => new HeaderParameter(provider: GroupsParameterProvider::class)])]
class Book {
    public string $id;
}
```

If you use Symfony but you don't have autoconfiguration enabled, declare the parameter as a tagged service:

```yaml
services:
  ApiPlatform\Tests\Fixtures\TestBundle\Parameter\CustomGroupParameterProvider:
    tags:
      - name: 'api_platform.parameter_provider'
        key: 'ApiPlatform\Tests\Fixtures\TestBundle\Parameter\CustomGroupParameterProvider'
```

or if you are using Laravel tag your provider with:

```php
<?php

namespace App\Providers;

use ApiPlatform\State\Provider\ParameterProvider;
use ApiPlatform\Tests\Fixtures\TestBundle\Parameter\CustomGroupParameterProvider;
use Illuminate\Support\ServiceProvider;

class AppServiceProvider extends ServiceProvider
{
    public function register(): void
    {
        $this->app->tag([CustomGroupParameterProvider::class], ParameterProvider::class);
    }
}
```

### Call a filter with Symfony

A Parameter can also call a filter and works on filters that impact the data persistence layer (Doctrine ORM, ODM and Eloquent filters are supported). Let's assume, that we have an Order filter declared:

```yaml
# config/services.yaml
services:
  offer.order_filter:
    parent: 'api_platform.doctrine.orm.order_filter'
    arguments:
      $properties: { id: ~, name: ~ }
      $orderParameterName: order
    tags: ['api_platform.filter']
```

```php
namespace App\ApiResource;

use ApiPlatform\Metadata\QueryParameter;

#[GetCollection(
    uriTemplate: 'orders',
    parameters: [
        'order[:property]' => new QueryParameter(filter: 'offer.order_filter'),
    ]
)
class Offer {
    public string $id;
    public string $name;
}
```

### Call a filter with Laravel

A Parameter can also call a filter and works on filters that impact the data persistence layer (Doctrine ORM, ODM and Eloquent filters are supported). Let's assume, that we have an Order filter declared:

```php
<?php

namespace App\Providers;

use ApiPlatform\Laravel\Eloquent\Filter\OrderFilter;
use ApiPlatform\Metadata\ApiFilter;
use Illuminate\Support\ServiceProvider;

class AppServiceProvider extends ServiceProvider
{
    $public function register(): void
    {
        $this->app->singleton(OrderFilter::class, function ($app) {
            return new OrderFilter(['id' => null, 'name' => null], 'order');
        });

        $this->app->tag([OrderFilter::class], ApiFilter::class);
    }
}
```


```php
namespace App\ApiResource;

use ApiPlatform\Metadata\QueryParameter;

#[GetCollection(
    uriTemplate: 'orders',
    parameters: [
        'order[:property]' => new QueryParameter(filter: 'offer.order_filter'),
    ]
)
class Offer {
    public string $id;
    public string $name;
}
```

### Header parameters

The `HeaderParameter` attribute allows to create a parameter that's using HTTP Headers instead of query parameters:

```php
namespace App\ApiResource;

use ApiPlatform\Metadata\HeaderParameter;
use App\Filter\MyApiKeyFilter;

#[HeaderParameter(key: 'API_KEY', filter: MyApiKeyFilter::class)]
class Book {
}
```

When you declare a parameter on top of a class, you need to specify it's key.

### The :property placeholder

When used on a Parameter, the `:property` placeholder allows to map automatically a parameter to the readable properties of your resource.

```php
namespace App\ApiResource;

use App\Filter\SearchFilter;
use ApiPlatform\Metadata\QueryParameter;

#[QueryParameter(key: ':property', filter: SearchFilter::class)]
class Book {
    public string $id;
    public string $title;
    public Author $author;
}
```

This will declare a query parameter for each property (ID, title and author) calling the SearchFilter.

This is especially useful for sort filters where you'd like to use `?sort[name]=asc`:

```php
namespace App\ApiResource;

use App\Filter\OrderFilter;
use ApiPlatform\Metadata\QueryParameter;

#[QueryParameter(key: 'sort[:property]', filter: OrderFilter::class)]
class Book {
    public string $id;
    public string $title;
    public Author $author;
}
```

### Documentation

A parameter is quite close to its documentation and you can specify the JSON Schema and/or the OpenAPI documentation:

```php
namespace App\ApiResource;

use ApiPlatform\Metadata\QueryParameter;
use ApiPlatform\OpenApi\Model\Parameter;

#[QueryParameter(
    key: 'q',
    required: true,
    schema: ['type' => 'string'],
    openApi: new Parameter(in: 'query', name: 'q', allowEmptyValue: true)
)]
class Book {
    public string $id;
    public string $title;
    public Author $author;
}
```

### Filter aliasing

Filter aliasing is done by declaring a parameter key with a different property:

```php
#[GetCollection(
    parameters: [
        'fooAlias' => new QueryParameter(filter: 'app_search_filter_via_parameter', property: 'foo'),
    ]
)]
class Book {
    public string $id;
    public string $foo;
}
```

If you need you can use the `filterContext` to transfer information between a parmameter and its filter.

### Parameter validation

Parameter validation is automatic based on the configuration for example:

```php
<?php
use ApiPlatform\Metadata\GetCollection;
use ApiPlatform\Metadata\QueryParameter;

#[GetCollection(
    uriTemplate: 'validate_parameters{._format}',
    parameters: [
        'enum' => new QueryParameter(schema: ['enum' => ['a', 'b'], 'uniqueItems' => true]),
        'num' => new QueryParameter(schema: ['minimum' => 1, 'maximum' => 3]),
        'exclusiveNum' => new QueryParameter(schema: ['exclusiveMinimum' => 1, 'exclusiveMaximum' => 3]),
        'blank' => new QueryParameter(openApi: new OpenApiParameter(name: 'blank', in: 'query', allowEmptyValue: false)),
        'length' => new QueryParameter(schema: ['maxLength' => 1, 'minLength' => 3]),
        'array' => new QueryParameter(schema: ['minItems' => 2, 'maxItems' => 3]),
        'multipleOf' => new QueryParameter(schema: ['multipleOf' => 2]),
        'pattern' => new QueryParameter(schema: ['pattern' => '/\d/']),
        'required' => new QueryParameter(required: true),
    ],
)]
class ValidateParameter {}
```

You can also use your own constraint by setting the `constraints` option on a Parameter. In that case we won't setup the automatic validation for you and it'll replace our defaults.

### Parameter security

Parameters may have security checks:

```php
<?php
use ApiPlatform\Metadata\GetCollection;
use ApiPlatform\Metadata\HeaderParameter;
use ApiPlatform\Metadata\QueryParameter;

#[GetCollection(
    uriTemplate: 'security_parameters{._format}',
    parameters: [
        'sensitive' => new QueryParameter(security: 'is_granted("ROLE_ADMIN")'),
        'auth' => new HeaderParameter(security: '"secretKey" == auth[0]'),
    ],
)]
class SecurityParameter {}
```

### Decorate a Doctrine filter using Symfony

A filter that implements the `ApiPlatform\Doctrine\Common\Filter\PropertyAwareFilterInterface` interface can be decorated:

```php
namespace App\Doctrine\Filter;

use ApiPlatform\Doctrine\Common\Filter\PropertyAwareFilterInterface;
use ApiPlatform\Doctrine\Orm\Filter\FilterInterface;
use ApiPlatform\Doctrine\Orm\Util\QueryNameGeneratorInterface;
use ApiPlatform\Metadata\Operation;
use Doctrine\ORM\QueryBuilder;
use Symfony\Component\DependencyInjection\Attribute\Autowire;

final class SearchTextAndDateFilter implements FilterInterface
{
    public function __construct(#[Autowire('@api_platform.doctrine.orm.search_filter.instance')] readonly FilterInterface $searchFilter, #[Autowire('@api_platform.doctrine.orm.date_filter.instance')] readonly FilterInterface $dateFilter, protected ?array $properties = null, private array $dateFilterProperties = [], private array $searchFilterProperties = [])
    {
    }

    // This function is only used to hook in documentation generators (supported by Swagger and Hydra)
    public function getDescription(string $resourceClass): array
    {
        if ($this->searchFilter instanceof PropertyAwareFilterInterface) {
            $this->searchFilter->setProperties($this->searchFilterProperties);
        }
        if ($this->dateFilter instanceof PropertyAwareFilterInterface) {
            $this->dateFilter->setProperties($this->dateFilterProperties);
        }

        return array_merge($this->searchFilter->getDescription($resourceClass), $this->dateFilter->getDescription($resourceClass));
    }

    public function apply(QueryBuilder $queryBuilder, QueryNameGeneratorInterface $queryNameGenerator, string $resourceClass, ?Operation $operation = null, array $context = []): void
    {
        if ($this->searchFilter instanceof PropertyAwareFilterInterface) {
            $this->searchFilter->setProperties($this->searchFilterProperties);
        }
        if ($this->dateFilter instanceof PropertyAwareFilterInterface) {
            $this->dateFilter->setProperties($this->dateFilterProperties);
        }

        $this->searchFilter->apply($queryBuilder, $queryNameGenerator, $resourceClass, $operation, ['filters' => $context['filters']['searchOnTextAndDate']] + $context);
        $this->dateFilter->apply($queryBuilder, $queryNameGenerator, $resourceClass, $operation, ['filters' => $context['filters']['searchOnTextAndDate']] + $context);
    }
}
```

This can be used with parameters using attributes:

```php
namespace App\Entity;

use ApiPlatform\Metadata\QueryParameter;

#[GetCollection(
    uriTemplate: 'search_filter_parameter{._format}',
    parameters: [
        'searchOnTextAndDate[:property]' => new QueryParameter(filter: 'app_filter_date_and_search'),
    ]
)]
// Note that we link the parameter filter and this filter using the "alias" option:
#[ApiFilter(SearchTextAndDateFilter::class, alias: 'app_filter_date_and_search', properties: ['foo', 'createdAt'], arguments: ['dateFilterProperties' => ['createdAt' => 'exclude_null'], 'searchFilterProperties' => ['foo' => 'exact']])]
#[ORM\Entity]
class SearchFilterParameter
{
    /**
     * @var int The id
     */
    #[ORM\Column(type: 'integer')]
    #[ORM\Id]
    #[ORM\GeneratedValue(strategy: 'AUTO')]
    private ?int $id = null;
    #[ORM\Column(type: 'string')]
    private string $foo = '';

    #[ORM\Column(type: 'datetime_immutable', nullable: true)]
    private ?\DateTimeImmutable $createdAt = null;

    public function getId(): ?int
    {
        return $this->id;
    }

    public function getFoo(): string
    {
        return $this->foo;
    }

    public function setFoo(string $foo): void
    {
        $this->foo = $foo;
    }

    public function getCreatedAt(): ?\DateTimeImmutable
    {
        return $this->createdAt;
    }

    public function setCreatedAt(\DateTimeImmutable $createdAt): void
    {
        $this->createdAt = $createdAt;
    }
}
```

## Serializer Filters

### Group Filter

> [!NOTE]
> In Symfony we use the term “entities”, while the following documentation is mostly for Laravel “models”.

The group filter allows you to filter by serialization groups.

Syntax: `?groups[]=<group>`

You can add as many groups as you need.

Enable the filter:

```php
<?php
// api/src/Entity/Book.php
namespace App\Entity;

use ApiPlatform\Metadata\ApiFilter;
use ApiPlatform\Metadata\ApiResource;
use ApiPlatform\Serializer\Filter\GroupFilter;

#[ApiResource]
#[ApiFilter(GroupFilter::class, arguments: ['parameterName' => 'groups', 'overrideDefaultGroups' => false, 'whitelist' => ['allowed_group']])]
class Book
{
    // ...
}
```

Three arguments are available to configure the filter:

- `parameterName` is the query parameter name (default `groups`)
- `overrideDefaultGroups` allows to override the default serialization groups (default `false`)
- `whitelist` groups whitelist to avoid uncontrolled data exposure (default `null` to allow all groups)

Given that the collection endpoint is `/books`, you can filter by serialization groups with the following query: `/books?groups[]=read&groups[]=write`.

### Property filter

**Note:** We strongly recommend using [Vulcain](https://vulcain.rocks) instead of this filter.
Vulcain is faster, allows a better hit rate, and is supported out of the box in the API Platform distribution.

The property filter adds the possibility to select the properties to serialize (sparse fieldsets).

Syntax: `?properties[]=<property>&properties[<relation>][]=<property>`

You can add as many properties as you need.

Enable the filter:

```php
<?php
// api/src/Entity/Book.php
namespace App\Entity;

use ApiPlatform\Metadata\ApiFilter;
use ApiPlatform\Metadata\ApiResource;
use ApiPlatform\Serializer\Filter\PropertyFilter;

#[ApiResource]
#[ApiFilter(PropertyFilter::class, arguments: ['parameterName' => 'properties', 'overrideDefaultProperties' => false, 'whitelist' => ['allowed_property']])]
class Book
{
    // ...
}
```

Three arguments are available to configure the filter:

- `parameterName` is the query parameter name (default `properties`)
- `overrideDefaultProperties` allows to override the default serialization properties (default `false`)
- `whitelist` properties whitelist to avoid uncontrolled data exposure (default `null` to allow all properties)

Given that the collection endpoint is `/books`, you can filter the serialization properties with the following query: `/books?properties[]=title&properties[]=author`.
If you want to include some properties of the nested "author" document, use: `/books?properties[]=title&properties[author][]=name`.

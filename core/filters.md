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
- [Doctrine filters documentation](../core/doctrine-filters.md)
- [Elasticsearch filters documentation](../core/elasticsearch-filters.md)
- [Laravel filters documentation](../laravel/filters.md)

## Parameters

You can declare parameters on a Resource or an Operation through the `parameters` property.

```php
namespace App\ApiResource;

use ApiPlatform\Metadata\GetCollection;
use ApiPlatform\Metadata\QueryParameter;

// This parameter "page" works only on /books
#[GetCollection(uriTemplate: '/books', parameters: ['page' => new QueryParameter])]
// This parameter is available on every operation, key is mandatory
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

If you use Symfony, but you don't have autoconfiguration enabled, declare the parameter as a tagged service:

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

We can use this filter specifying we want a query parameter with the `:property` placeholder:

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

We can use this filter specifying we want a query parameter with the `:property` placeholder:

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

A parameter is quite close to its documentation, and you can specify the JSON Schema and/or the OpenAPI documentation:

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

If you use Laravel refers to the [Laravel Validation documentation](../laravel/validation.md).

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

You can also use your own constraint by setting the `constraints` option on a Parameter. In that case we won't set up the automatic validation for you, and it'll replace our defaults.

### Parameter security

If you use Laravel refers to the [Laravel Security documentation](../laravel/security.md).

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

## Serializer Filters

### Group Filter

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

## Creating Custom Filters

Custom filters can be written by implementing the `ApiPlatform\Api\FilterInterface` interface.

API Platform provides a convenient way to create Doctrine ORM and MongoDB ODM filters. If you use [custom state providers](state-providers.md),
you can still create filters by implementing the previously mentioned interface, but - as API Platform isn't aware of your
persistence system's internals - you have to create the filtering logic by yourself.

If you need more information about creating custom filters, refer to the following documentation:

- [Creating Custom Doctrine ORM filters](../core/doctrine-filters.md#creating-custom-doctrine-orm-filters)
- [Creating Custom Doctrine Mongo ODM filters](../core/doctrine-filters.md#creating-custom-doctrine-mongodb-odm-filters)
- [Creating Custom Doctrine ORM filters](../core/elasticsearch-filters.md#creating-custom-elasticsearch-filters)

## ApiFilter Attribute

The attribute can be used on a `property` or on a `class`.

If the attribute is given over a property, the filter will be configured on the property. For example, let's add a search filter on `name` and on the `prop` property of the `colors` relation:

```php
<?php
// api/src/Entity/DummyCar.php
namespace App\Entity;

use ApiPlatform\Metadata\ApiFilter;
use ApiPlatform\Metadata\ApiResource;
use ApiPlatform\Doctrine\Orm\Filter\SearchFilter;
use Doctrine\Common\Collections\ArrayCollection;
use Doctrine\Common\Collections\Collection;
use Doctrine\ORM\Mapping as ORM;
use App\Entity\DummyCarColor;

#[ApiResource]
class DummyCar
{
    #[ORM\Id, ORM\Column, ORM\GeneratedValue]
    private ?int $id = null;

    #[ORM\Column]
    #[ApiFilter(SearchFilter::class, strategy: 'partial')]
    public ?string $name = null;

    #[ORM\OneToMany(mappedBy: "car", targetEntity: DummyCarColor::class)]
    #[ApiFilter(SearchFilter::class, properties: ['colors.prop' => 'ipartial'])]
    public Collection $colors;

    public function __construct()
    {
        $this->colors = new ArrayCollection();
    }

    // ...
}
```

On the first property, `name`, it's straightforward. The first attribute argument is the filter class, the second specifies options, here, the strategy:

```php
#[ApiFilter(SearchFilter::class, strategy: 'partial')]
```

In the second attribute, we specify `properties` to which the filter should apply. It's necessary here because we don't want to filter `colors` but the `prop` property of the `colors` association.
Note that for each given property we specify the strategy:

```php
#[ApiFilter(SearchFilter::class, properties: ['colors.prop' => 'ipartial'])]
```

The `ApiFilter` attribute can be set on the class as well. If you don't specify any properties, it'll act on every property of the class.

For example, let's define three data filters (`DateFilter`, `SearchFilter` and `BooleanFilter`) and two serialization filters (`PropertyFilter` and `GroupFilter`) on our `DummyCar` class:

```php
<?php
// api/src/Entity/DummyCar.php
namespace App\Entity;

use ApiPlatform\Doctrine\Common\Filter\DateFilterInterface;
use ApiPlatform\Metadata\ApiFilter;
use ApiPlatform\Metadata\ApiResource;
use ApiPlatform\Doctrine\Orm\Filter\BooleanFilter;
use ApiPlatform\Doctrine\Orm\Filter\DateFilter;
use ApiPlatform\Doctrine\Orm\Filter\SearchFilter;
use ApiPlatform\Serializer\Filter\GroupFilter;
use ApiPlatform\Serializer\Filter\PropertyFilter;
use Doctrine\ORM\Mapping as ORM;

#[ApiResource]
#[ApiFilter(BooleanFilter::class)]
#[ApiFilter(DateFilter::class, strategy: DateFilterInterface::EXCLUDE_NULL)]
#[ApiFilter(SearchFilter::class, properties: ['colors.prop' => 'ipartial', 'name' => 'partial'])]
#[ApiFilter(PropertyFilter::class, arguments: ['parameterName' => 'foobar'])]
#[ApiFilter(GroupFilter::class, arguments: ['parameterName' => 'foobargroups'])]
class DummyCar
{
    // ...
}

```

The `BooleanFilter` is applied to every `Boolean` property of the class. Indeed, in each core filter, we check the Doctrine type. It's written only by using the filter class:

```php
#[ApiFilter(BooleanFilter::class)]
```

The `DateFilter` given here will be applied to every `Date` property of the `DummyCar` class with the `DateFilterInterface::EXCLUDE_NULL` strategy:

```php
#[ApiFilter(DateFilter::class, strategy: DateFilterInterface::EXCLUDE_NULL)]
```

The `SearchFilter` here adds properties. The result is the exact same as the example with attributes on properties:

```php
#[ApiFilter(SearchFilter::class, properties: ['colors.prop' => 'ipartial', 'name' => 'partial'])]
```

Note that you can specify the `properties` argument on every filter.

The next filters are not related to how the data is fetched but rather to how the serialization is done on those. We can give an `arguments` option ([see here for the available arguments](#serializer-filters)):

```php
#[ApiFilter(PropertyFilter::class, arguments: ['parameterName' => 'foobar'])]
#[ApiFilter(GroupFilter::class, arguments: ['parameterName' => 'foobargroups'])]
```

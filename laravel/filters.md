# Parameters and Filters

API Platform is great for Rapid Application Development and provides lots of functionalities out of the box such as collection filtering with Eloquent. Most of the filtering is done using query parameters, which are automatically documented and validated. If needed you can use [state providers](core/state-providers) or a [Links Handler] to provide data.

## Parameters

A filter is usually used via a `ApiPlatform\Metadata\QueryParameter` and is also available through `ApiPlatform\Metadata\HeaderParameter`. For example, let's declare an `EqualsFilter` on our `Book` to be able to query an exact match using `/books?name=Animal Farm. A Fairy Story`:

```php
// app/Models/Book.php 

use ApiPlatform\Laravel\Eloquent\Filter\EqualsFilter;
use ApiPlatform\Metadata\ApiResource;
use ApiPlatform\Metadata\QueryParameter;

#[ApiResource]
#[QueryParameter(key: 'name', filter: EqualsFilter::class)]
class Book extends Model
{
}
```

The `key` option specifies the query parameter and the `filter` applies the given value to a where clause:

```php
namespace ApiPlatform\Laravel\Eloquent\Filter;

use ApiPlatform\Metadata\Parameter;
use Illuminate\Database\Eloquent\Builder;
use Illuminate\Database\Eloquent\Model;

final class EqualsFilter implements FilterInterface
{
    /**
     * @param Builder<Model>       $builder
     * @param array<string, mixed> $context
     */
    public function apply(Builder $builder, mixed $values, Parameter $parameter, array $context = []): Builder
    {
        return $builder->where($parameter->getProperty(), $values);
    }
}
```

You can create your own filters by implementing the `ApiPlatform\Laravel\Eloquent\Filter\FilterInterface`. API Platform provides several eloquent filters for a RAD approach.

### Parameter Validation

You can add [validation rules](https://laravel.com/docs/validation) to parameters within the `constraints` attribute:

```php
// app/Models/Book.php 

use ApiPlatform\Laravel\Eloquent\Filter\PartialSearchFilter;
use ApiPlatform\Metadata\ApiResource;
use ApiPlatform\Metadata\QueryParameter;

#[ApiResource]
#[QueryParameter(key: 'name', filter: PartialSearchFilter::class, constraints: 'min:2')]
class Book extends Model
{
}
```

<!-- TODO: The `security` option also allows policy but we need to test this -->

### The `:property` Placeholder

When programming APIs you may need to apply a filter on many properties at once. For example, we're allowing to sort on every property of our ApiResource with a partial search filter:

```php
// app/Models/Book.php 

use ApiPlatform\Laravel\Eloquent\Filter\PartialSearchFilter;
use ApiPlatform\Laravel\Eloquent\Filter\OrderFilter;
use ApiPlatform\Metadata\ApiResource;
use ApiPlatform\Metadata\QueryParameter;

#[ApiResource]
#[QueryParameter(key: 'sort[:property]', filter: OrderFilter::class)]
#[QueryParameter(key: ':property', filter: PartialSearchFilter::class)]
class Book extends Model
{
}
```

The documentation will output a query parameter per property that applies the `PartialSearchFilter` and also gives the ability to sort by name and id using: `/books?name=search&order[id]=asc&order[name]=desc`.

## Filters

### Text

As shown above the following search filters are available: 

- `ApiPlatform\Laravel\Eloquent\Filter\PartialSearchFilter` queries `LIKE %term%`
- `ApiPlatform\Laravel\Eloquent\Filter\EqualsFilter` queries `= term`
- `ApiPlatform\Laravel\Eloquent\Filter\StartSearchFilter` queries `LIKE term%`
- `ApiPlatform\Laravel\Eloquent\Filter\EndSearchFilter` queries `LIKE %term`

### Date

The `DateFilter` allows to filter dates with an operator (`eq`, `lt`, `gt`, `lte`, `gte`): 

```php
// app/Models/Book.php 

use ApiPlatform\Laravel\Eloquent\Filter\DateFilter;

#[ApiResource]
#[QueryParameter(key: 'publicationDate', filter: DateFilter::class, filterContext: ['include_nulls' => true])]
class Book extends Model
{
    use HasUlids;

    public function author(): BelongsTo
    {
        return $this->belongsTo(Author::class);
    }
}
```

Our default strategy is to exclude null values, just remove the `filterContext` if you want to exclude nulls. 

### Or

The `OrFilter` allows to filter using an `OR WHERE` clause:

```php
// app/Models/Book.php 

use ApiPlatform\Laravel\Eloquent\Filter\DateFilter;

#[ApiResource]
#[QueryParameter(
    key: 'q',
    filter: new OrFilter(new EqualsFilter()),
    property: 'isbn'
)]
class Book extends Model
{
    use HasUlids;

    public function author(): BelongsTo
    {
        return $this->belongsTo(Author::class);
    }
}
```

This allows to query multiple `isbn` values with a `q` query parameter: `/books?q[]=9781784043735&q[]=9780369406361`.

<!-- ### Number

TODO -->

### PropertyFilter

Note: We strongly recommend using [Vulcain](https://vulcain.rocks) instead of this filter. Vulcain is faster, allows a better hit rate, and is supported out of the box in the API Platform distribution.

The property filter adds the possibility to select the properties to serialize (sparse fieldsets).

```php
// app/Models/Book.php 

use ApiPlatform\Laravel\Eloquent\Filter\DateFilter;
use ApiPlatform\Serializer\Filter\PropertyFilter;

#[ApiResource]
#[QueryParameter(key: 'properties', filter: PropertyFilter::class)]
class Book extends Model
{
    use HasUlids;

    public function author(): BelongsTo
    {
        return $this->belongsTo(Author::class);
    }
}
```

A few `filterContext` options are available to configure the filter:

* `override_default_properties` allows to override the default serialization properties (default `false`) Using `true` is dangerous, use carefully this can expose unwanted data!
* `whitelist` properties whitelist to avoid uncontrolled data exposure (default `null` to allow all properties)

Given that the collection endpoint is `/books`, you can filter the serialization properties with the following query: `/books?properties[]=title&properties[]=author`.

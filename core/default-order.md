# Overriding Default Order

API Platform provides an easy way to override the default order of items in your collection.

By default, items in the collection are ordered in ascending (ASC) order by their resource
identifier(s). If you want to customize this order, you must add an `order` attribute on your
ApiResource annotation:

<code-selector>

```php
<?php
// api/src/ApiResource/Book.php with Symfony or app/ApiResource/Book.php with Laravel
namespace App\ApiResource;

use ApiPlatform\Metadata\ApiResource;

#[ApiResource(order: ['foo' => 'ASC'])]
class Book
{
    // ...

    /**
     * ...
     */
    public $foo;

    // ...
}
```

```yaml
# api/config/api_platform/resources/Book.yaml
# The YAML syntax is only supported for Symfony
App\ApiResource\Book:
    order:
        foo: ASC
```

</code-selector>

This `order` attribute is used as an array: the key defines the order field, the values defines the
direction. If you only specify the key, `ASC` direction will be used as default. For example, to
order by `foo` & `bar`:

<code-selector>

```php
<?php
// api/src/ApiResource/Book.php with Symfony or app/ApiResource/Book.php with Laravel
namespace App\ApiResource;

use ApiPlatform\Metadata\ApiResource;

#[ApiResource(order: ['foo', 'bar'])]
class Book
{
    // ...

    /**
     * ...
     */
    public $foo;

    /**
     * ...
     */
    public $bar;

    // ...
}
```

```yaml
# api/config/api_platform/resources/Book.yaml
# The YAML syntax is only supported for Symfony
App\ApiResource\Book:
    order: ["foo", "bar"]
```

</code-selector>

It's also possible to configure the default order on an association property:

<code-selector>

```php
<?php
// api/src/ApiResource/Book.php with Symfony or app/ApiResource/Book.php with Laravel
namespace App\ApiResource;

use ApiPlatform\Metadata\ApiResource;

#[ApiResource(order: ['author.username'])]
class Book
{
    // ...

    /**
     * @var User
     */
    public $author;

    // ...
}
```

```yaml
# api/config/api_platform/resources/Book.yaml
# The YAML syntax is only supported for Symfony
App\ApiResource\Book:
    order: ["author.username"]
```

</code-selector>

Another possibility is to apply the default order for a specific collection operation.

<code-selector>

```php
<?php
// api/src/ApiResource/Book.php with Symfony or app/ApiResource/Book.php with Laravel
namespace App\ApiResource;

use ApiPlatform\Metadata\GetCollection;
use ApiPlatform\Metadata\ApiResource;

#[ApiResource(operations: [
    new GetCollection(),
    new GetCollection(name: 'get_desc_custom', uriTemplate: 'custom_collection_desc_foos', order: ['name' => 'DESC'])],
    new GetCollection(name: 'get_asc_custom', uriTemplate: 'custom_collection_asc_foos', order: ['name' => 'ASC'])]
])]
class Book
{
    // ...

    /**
     * @var string
     */
    public $name;

    // ...
}
```

```yaml
# api/config/api_platform/resources/Book.yaml
# The YAML syntax is only supported for Symfony
App\ApiResource\Book:
    ApiPlatform\Metadata\GetCollection: ~
    get_desc_custom:
        class: ApiPlatform\Metadata\GetCollection
        uriTemplate: custom_collection_desc_foos
        order:
            name: DESC
    get_asc_custom:
        class: ApiPlatform\Metadata\GetCollection
        uriTemplate: custom_collection_asc_foos
        order:
            name: ASC
```

</code-selector>

## Global Default Order and Operation Order Precedence

`api_platform.defaults.order` (set under the `defaults:` key in `api_platform.yaml`) applies an
order to every operation. It is not a fallback that yields to operation-level configuration — it is
a cross-cutting invariant that is always applied.

When an operation also defines its own `order`, the two arrays are **merged**: the global default
keys come first, followed by the operation-level keys. This means the global default takes priority
in the SQL `ORDER BY` clause, and the operation-level keys act as tie-breakers.

For example, given this configuration:

```yaml
# config/packages/api_platform.yaml
api_platform:
    defaults:
        order:
            createdAt: DESC
```

And this resource:

```php
<?php
// api/src/ApiResource/Book.php
namespace App\ApiResource;

use ApiPlatform\Metadata\ApiResource;
use ApiPlatform\Metadata\GetCollection;

#[ApiResource(operations: [
    new GetCollection(order: ['title' => 'ASC']),
])]
class Book
{
    public \DateTimeImmutable $createdAt;
    public string $title;
}
```

The effective order for the `GetCollection` operation is
`['createdAt' => 'DESC', 'title' => 'ASC']`, producing `ORDER BY createdAt DESC, title ASC`. The
global default is prepended to the operation-level order, not replaced by it.

This behavior is intentional. `api_platform.defaults.order` is designed for invariants such as
"always order by `createdAt DESC`" that must hold across all collections regardless of which
operation is called.

**If you want an operation to use a specific order with no global keys prepended**, do not set
`api_platform.defaults.order`. Instead, set the `order` explicitly on each operation or resource
where you need it. For more control over ordering logic, implement a custom
[Doctrine ORM extension](extensions.md) that replaces the built-in `OrderExtension`.

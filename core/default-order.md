# Overriding Default Order

API Platform Core provides an easy way to override the default order of items in your collection.

By default, items in the collection are ordered in ascending (ASC) order by their resource identifier(s). If you want to
customize this order, you must add an `order` attribute on your ApiResource annotation:

[codeSelector]

```php
<?php
// api/src/Entity/Book.php
namespace App\Entity;

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
App\Entity\Book:
    order:
        foo: ASC
```

[/codeSelector]

This `order` attribute is used as an array: the key defines the order field, the values defines the direction.
If you only specify the key, `ASC` direction will be used as default. For example, to order by `foo` & `bar`:

[codeSelector]

```php
<?php
// api/src/Entity/Book.php
namespace App\Entity;

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
App\Entity\Book:
    order: ['foo', 'bar']
```

[/codeSelector]

It's also possible to configure the default order on an association property:

[codeSelector]

```php
<?php
// api/src/Entity/Book.php
namespace App\Entity;

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
App\Entity\Book:
    order: ['author.username']
```

[/codeSelector]

Another possibility is to apply the default order for a specific collection operation, which will override the global default order configuration.

[codeSelector]

```php
<?php
// api/src/Entity/Book.php
namespace App\Entity;

use ApiPlatform\Metadata\GetCollection;
use ApiPlatform\Metadata\ApiResource;

#[ApiResource]
#[GetCollection]
#[GetCollection(name: 'get_desc_custom', uriTemplate: 'custom_collection_desc_foos', order: ['name' => 'DESC'])]
#[GetCollection(name: 'get_asc_custom', uriTemplate: 'custom_collection_asc_foos', order: ['name' => 'ASC'])]
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
App\Entity\Book:
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

[/codeSelector]

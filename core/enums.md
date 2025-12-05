# Enums as API Resources

API Platform provides support for PHP 8.1+ `BackedEnum`s, allowing them to be exposed as first-class
API resources. This enables clients to discover available enum cases and their associated metadata
directly through your API.

## Exposing BackedEnums

To expose a `BackedEnum` as an API resource, simply apply the `#[ApiResource]` attribute to your
enum class:

```php
<?php
// api/src/Enum/Status.php
namespace App\Enum;

use ApiPlatform\Metadata\ApiResource;

#[ApiResource]
enum Status: int
{
    case DRAFT = 0;
    case PUBLISHED = 1;
    case ARCHIVED = 2;
}
```

By default, API Platform will automatically generate `GET` and `GET Collection` operations for your
enum resource. The enum's `value` will be used as the identifier for individual enum cases.

### Default Operations and Identifiers

- **Collection**: `GET /statuses` will return a collection of all enum cases.
- **Item**: `GET /statuses/{value}` will return a single enum case based on its `value`.

Example `GET /statuses` response:

```json
[
    {
        "name": "DRAFT",
        "value": 0
    },
    {
        "name": "PUBLISHED",
        "value": 1
    },
    {
        "name": "ARCHIVED",
        "value": 2
    }
]
```

Example `GET /statuses/1` response:

```json
{
    "name": "PUBLISHED",
    "value": 1
}
```

### Customizing Enum Resources

You can customize the behavior of enum resources using standard API Platform attributes.

#### Custom Identifier

If you wish to use a property other than `value` as the identifier, or to expose additional data,
you can implement methods within your enum and mark them with `#[ApiProperty]`. For instance, to use
the enum `name` as an identifier, you can implement `getId()`:

```php
<?php
// api/src/Enum/Audit.php
namespace App\Enum;

use ApiPlatform\Metadata\ApiProperty;
use ApiPlatform\Metadata\ApiResource;

#[ApiResource]
enum Audit: string
{
    case Pending = 'pending';
    case Passed = 'passed';
    case Failed = 'failed';

    #[ApiProperty(identifier: true)]
    public function getId(): string
    {
        return $this->name;
    }
}
```

#### Adding Custom Properties

You can add custom properties to your enum resource by defining public methods and marking them with
`#[ApiProperty]`:

```php
<?php
// api/src/Enum/Status.php
namespace App\Enum;

use ApiPlatform\Metadata\ApiProperty;
use ApiPlatform\Metadata\ApiResource;

#[ApiResource]
enum Status: int
{
    case DRAFT = 0;
    case PUBLISHED = 1;
    case ARCHIVED = 2;

    #[ApiProperty]
    public function getDescription(): string
    {
        return match ($this) {
            self::DRAFT => 'Article is not ready for public consumption',
            self::PUBLISHED => 'Article is publicly available',
            self::ARCHIVED => 'Article content is outdated or superseded',
        };
    }
}
```

With the above, `GET /statuses/0` might return:

```json
{
    "name": "DRAFT",
    "value": 0,
    "description": "Article is not ready for public consumption"
}
```

#### Custom State Providers

For more advanced customization, you can implement custom state providers for your enum resources. A
common pattern is to use a trait to provide common functionality:

```php
<?php
// api/src/Enum/EnumApiResourceTrait.php
namespace App\Enum;

use ApiPlatform\Metadata\Operation;
use BackedEnum;

trait EnumApiResourceTrait
{
    public function getId(): string|int
    {
        return $this->value;
    }

    public function getValue(): int|string
    {
        return $this->value;
    }

    public static function getCases(): array
    {
        return self::cases();
    }

    public static function getCase(Operation $operation, array $uriVariables): ?BackedEnum
    {
        $id = is_numeric($uriVariables['id']) ? (int) $uriVariables['id'] : $uriVariables['id'];

        return array_reduce(self::cases(), static fn($c, BackedEnum $case) => $case->name === $id || $case->value === $id ? $case : $c, null);
    }
}
```

Then, apply the trait and specify the providers in your enum:

```php
<?php
// api/src/Enum/Audit.php
namespace App\Enum;

use ApiPlatform\Metadata\ApiResource;
use ApiPlatform\Metadata\Get;
use ApiPlatform\Metadata\GetCollection;

#[ApiResource]
#[GetCollection(provider: Audit::class.'::getCases')]
#[Get(provider: Audit::class.'::getCase')]
enum Audit: string
{
    use EnumApiResourceTrait;

    case Pending = 'pending';
    case Passed = 'passed';
    case Failed = 'failed';
}
```

### Enums as Property Values

When an enum is used as a property in another `ApiResource`, it will be serialized by its `value` by
default.

Consider an `Article` resource with a `Status` enum property:

```php
<?php
// api/src/ApiResource/Article.php
namespace App\ApiResource;

use ApiPlatform\Metadata\ApiResource;
use App\Enum\Status; // Import your enum

#[ApiResource]
class Article
{
    public ?int $id = null;
    public ?string $title = null;
    public ?Status $status = null; // Enum property
}
```

The serialization of `Article` will include the `value` of the `Status` enum:

```json
{
    "id": 1,
    "title": "Once Upon A Title",
    "status": 1
}
```

The OpenAPI schema will also correctly represent the enum as its backing type (`integer` for
`Status`):

```json
{
    "Article": {
        "type": "object",
        "properties": {
            "id": {
                "readOnly": true,
                "type": "integer"
            },
            "title": {
                "type": "string"
            },
            "status": {
                "type": "integer",
                "enum": [0, 1, 2] // Enum values derived from the BackedEnum
            }
        }
    }
}
```

### Referencing Enum Resources as Property Values

If you have exposed an enum as an `ApiResource` (e.g., `#[ApiResource]` on `Status` enum), and then
use that enum as a property in another resource (e.g., `Article::$status`), API Platform will
serialize it as an IRI (Internationalized Resource Identifier) by default.

```php
<?php
// api/src/ApiResource/Article.php
namespace App\ApiResource;

use ApiPlatform\Metadata\ApiResource;
use App\Enum\Status; // Assume Status is also an ApiResource

#[ApiResource]
class Article
{
    public ?int $id = null;
    public ?string $title = null;
    public ?Status $status = null; // This will now be an IRI
}
```

The serialization of `Article` will include an IRI for the `Status` enum:

```json
{
    "id": 1,
    "title": "Once Upon A Title",
    "status": "/statuses/1"
}
```

If you prefer the enum to be serialized by its `value` instead of an IRI, even when it's an
`ApiResource`, you might need to adjust your serialization context or create a custom normalizer if
the default behavior doesn't suit your needs.

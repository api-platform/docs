# Static Code Tables

Almost every application uses one or more sets of codes that typically consist of a code that is stored on records in the
database and a description that explains the meaning of the code. More often than not, these codes remain static over the
lifetime of the application.

The question with an API is, how does one expose these codes via the standard API operations without storing them in special
code database tables.

There is an elegant way to expose codes, in three easy steps, as if they were stored in the database, and as if there were
database relations between the data tables and the virtual "code tables".

## Using Static Arrays

Step 1 is to create an API resource for the virtual "code table":

```php
<?php
// api/src/Entity/ItemType.php

namespace App\Entity;

use ApiPlatform\Core\Annotation\ApiProperty;
use ApiPlatform\Core\Annotation\ApiResource;
use Symfony\Component\Serializer\Annotation\Groups;

#[ApiResource(
    collectionOperations: ['get'],
    itemOperations      : ['get'],
    normalizationContext: [
        'groups' => [self::ITEM_TYPE_READ]
    ],
)]
class ItemType
{
    public const ITEM_TYPE_READ = 'item_type:read';

    // This is your "code table"
    private static array $data = [
        'foo' => 'A foo type of item',
        'bar' => 'A bar type of item',
    ];

    #[Groups([self::ITEM_TYPE_READ])]
    #[ApiProperty(identifier: true)]
    public ?string $id = null;

    #[Groups([self::ITEM_TYPE_READ])]
    public ?string $description = null;

    public function __construct(?string $id, ?string $description) {
        $this->id = $id;
        $this->description = $description;
    }

    public static function createInstance(string $id): ?self
    {
        $id = self::extractIdFromIri($id);
        if (!isset(self::$data[$id])) {
            return null;
        }

        return new self($id, self::$data[$id]);
    }

    /**
     * @return ItemType[]
     */
    public static function getCollection(): array
    {
        $collection = [];
        foreach (array_keys(self::$data) as $id) {
            $collection[] = self::createInstance($id);
        }

        return $collection;
    }

    public static function getItem(string $id): ?self
    {
        return self::createInstance($id);
    }

    private static function extractIdFromIri(string $iri): ?string
    {
        if (str_ends_with($iri, '/')) {
            $iri = substr($iri, 0, -1);
        }
        $iriParts = explode('/', $iri);

        return array_pop($iriParts);
    }
}
```

Step 2 is to create a custom data provider for the "code table" API resource:

```php
<?php
// api/src/DataProvider/ItemTypeDataProvider.php

namespace App\DataProvider;

use ApiPlatform\Core\DataProvider\CollectionDataProviderInterface;
use ApiPlatform\Core\DataProvider\ItemDataProviderInterface;
use ApiPlatform\Core\DataProvider\RestrictedDataProviderInterface;
use App\Entity\ItemType;

class ItemTypeDataProvider implements CollectionDataProviderInterface, ItemDataProviderInterface, RestrictedDataProviderInterface
{
    public function getCollection(
        string $resourceClass,
        string $operationName = null
    ): array {
        return ItemType::getCollection();
    }

    public function getItem(
        string $resourceClass,
        $id,
        string $operationName = null,
        array $context = []
    ): ?ItemType {
        return ItemType::getItem($id);
    }

    public function supports(
        string $resourceClass,
        string $operationName = null,
        array $context = []
    ): bool {
        return ItemType::class === $resourceClass;
    }
}
```

At this point of the process the API will have functioning collection GET and item GET endpoints where you can obtain the
details of the codes and their descriptions as if they were stored in the database.

Step 3 is to simulate a database relation between the data table and this virtual code table:

```php
<?php
// api/src/Entity/Item.php

namespace App\Entity;

use ApiPlatform\Core\Annotation\ApiResource;
use Doctrine\ORM\Mapping as ORM;

#[ORM\Entity]
#[ApiResource(
    denormalizationContext: [
        'groups' => [self::ITEM_WRITE]
    ],
     normalizationContext  : [
        'groups' => [self::ITEM_READ]
    ]
)]
class Item
{
    public const ITEM_READ = 'item:read';
    public const ITEM_WRITE = 'item:write';

    // All the properties of the entity

    #[ORM\Column(type: 'string')]
    private ?string $itemTypeId = null;

    // All the methods of the entity

    #[Groups([self::ITEM_READ])]
    public function getItemType(): ItemType
    {
        return ItemType::getItem($this->itemTypeId);
    }

    #[Groups([self::ITEM_WRITE])]
    public function setItemType(ItemType $itemType): self
    {
        $this->itemTypeId = $itemType->id;

        return $this;
    }

}
```

Voila! It will now appear as if the codes are stored in standard database tables, but without the overhead of database access.

This solution is optimal for codes that rarely or never change. Code sets that are frequently modified are better served using
standard database tables and maintenance functions in the application codebase.

## Using PHP Enumerations

If you want to use the PHP `enum` type, do the following:

Replace the `ItemType` class from above with a new backed `enum` class:

```php
<?php
// api/src/Entity/ItemType.php

namespace App\Entity;

use ApiPlatform\Core\Annotation\ApiProperty;
use ApiPlatform\Core\Annotation\ApiResource;
use Symfony\Component\Serializer\Annotation\Groups;

#[ApiResource(
    collectionOperations: ['get'],
    itemOperations      : ['get'],
    normalizationContext: [
        'groups' => [self::ITEM_TYPE_READ]
    ],
)]
enum ItemType:string
{
    case Foo = 'foo';
    case Bar = 'bar';

    public const ITEM_TYPE_READ = 'item_type:read';

    public static function createInstance(string $id): ?self
    {
        $id = self::extractIdFromIri($id);
        try {
            return self::from($id);
        } catch (\ValueError) {
            return null;
        }
    }

    /**
     * @return ItemType[]
     */
    public static function getCollection(): array
    {
        $collection = [];
        foreach (self::cases() as $case) {
            $collection[] = self::createInstance(
                $case->value,
            );
        }

        return $collection;
    }

    public static function getItem(string $id): ?self
    {
        return self::createInstance($id);
    }

    #[Groups([self::ITEM_TYPE_READ])]
    public function getDescription(): string
    {
        return $this->label();
    }

    #[Groups([self::ITEM_TYPE_READ])]
    #[ApiProperty(identifier: true)]
    public function getId(): string
    {
        return $this->value;
    }

    public function label(): string
    {
        return match ($this) {
            self::Foo => 'A foo type of item',
            self::Bar => 'A bar type of item',
        };
    }

    private static function extractIdFromIri(string $iri): ?string
    {
        if (str_ends_with($iri, '/')) {
            $iri = substr($iri, 0, -1);
        }
        $iriParts = explode('/', $iri);

        return array_pop($iriParts);
    }

}
```

The `Item` and `ItemTypeDataProvider` classes remain exactly the same as above.

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

    public static function createInstance(?string $id): ?self
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

    public static function getItem(?string $id): ?self
    {
        return self::createInstance($id);
    }

    private static function extractIdFromIri(string $iri): ?string
    {
        $iri = (string) $iri; 
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
use Symfony\Component\Validator\Constraints as Assert;

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
    #[Assert\NotBlank]
    public function getItemType(): ?ItemType
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

    public static function createInstance(?string $id): ?self
    {
        $id = self::extractIdFromIri($id);
        return self::tryFrom($id);
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

    public static function getItem(?string $id): ?self
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

    private static function extractIdFromIri(?string $iri): ?string
    {
        $iri = (string) $iri;
        if (str_ends_with($iri, '/')) {
            $iri = substr($iri, 0, -1);
        }
        $iriParts = explode('/', $iri);

        return array_pop($iriParts);
    }

}
```

Create the `Item` class:

```php
<?php
// api/src/Entity/Item.php

namespace App\Entity;

use ApiPlatform\Core\Annotation\ApiResource;
use Doctrine\ORM\Mapping as ORM;
use Symfony\Component\Validator\Constraints as Assert;

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

    #[Groups([self::ITEM_READ, self::ITEM_WRITE])]
    #[Assert\NotBlank]
    #[ORM\Column(type: 'string', enumType: ItemType::class)]
    private ?ItemType $itemType = null;

    // All the methods of the entity

    public function getItemType(): ?ItemType
    {
        return $this->itemType;
    }

    public function setItemType(ItemType $itemType): self
    {
        $this->itemType = $itemType;

        return $this;
    }

}
```

The `ItemTypeDataProvider` class remains exactly the same as above.

## Simulating a Many-To-Many Code Data Relationship

Sometimes you need to create a many-to-many relationship between the data entity and the code entity.

That can be implemented in six easy steps.

Step 1 is to create an `ItemType` API resource for the virtual "code table" in exactly the same manner as described in
`Using PHP Enumerations` above.

Step 2 is to create an `ItemTypeDataProvider` custom data provider for the `ItemType` "code table" API resource as
described in `Using Static Arrays` above.

Step 3 is to simulate a many-to-many database relation between the data table and the `ItemType` virtual code table:

```php
<?php
// api/src/Entity/Item.php

namespace App\Entity;

use ApiPlatform\Core\Annotation\ApiResource;
use Doctrine\ORM\Mapping as ORM;
use Doctrine\Common\Collections\Collection;

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

    #[ORM\Column(type: 'json', nullable=true)]
    private ?array $itemTypeIds = null;

    #[Groups([self::ITEM_READ, self::ITEM_WRITE])]
    private ?Collection $itemTypes = null;

    public function __construct()
    {
        $this->initializeItemTypes();
    }

    // All the methods of the entity

    public function initializeItemTypes(): void
    {
        $this->itemTypes = new ArrayCollection();
    }

    public function getItemTypeIds(): array
    {
        return $this->itemTypeIds ?? [];
    }

    public function setItemTypeIds(?array $itemTypeIds): self
    {
        $this->itemTypeIds = $itemTypeIds;

        return $this;
    }

    public function addItemType(ItemType $itemType): self
    {
        if (!$this->itemTypes->contains($itemType)) {
            $this->itemTypes[] = $itemType;
        }

        return $this;
    }

    /**
     * @return Collection<ItemType>
     */
    public function getItemTypes(): Collection
    {
        return $this->itemTypes;
    }

    public function removeItemType(ItemType $itemType): self
    {
        $this->itemTypes->removeElement($itemType);

        return $this;
    }
}
```

Step 4 is to create a custom data persister for the `Item` entity:

```php
<?php
// api/src/DataPersister/ItemDataPersister.php

namespace App\DataPersister;

use ApiPlatform\Core\DataPersister\DataPersisterInterface;
use App\Entity\Item;
use App\Entity\ItemType;

class ItemDataPersister implements DataPersisterInterface
{
    public function __construct(private DataPersisterInterface $decoratedDoctrineDataPersister) {
    }

    /**
     * @param Item $data
     */
    public function persist($data): void
    {
        $itemTypeIds = [];
        /** @var ItemType $itemType */
        foreach ($data->getItemTypes() as $itemType) {
            $itemTypeIds[] = $itemType->value;
        }
        $data->setItemTypeIds($itemTypeIds);
        $this->decoratedDoctrineDataPersister->persist($data);
    }

    /**
     * @param Item $data
     */
    public function remove($data): void
    {
        $this->decoratedDoctrineDataPersister->remove($data);
    }

    public function supports($data): bool
    {
        return $data instanceof Item;
    }
```

Step 5 is to create a custom data provider for the `Item` entity:

```php
<?php
// api/src/DataProvider/ItemDataProvider.php

namespace App\DataProvider;

use ApiPlatform\Core\Bridge\Doctrine\Orm\CollectionDataProvider;
use ApiPlatform\Core\Bridge\Doctrine\Orm\ItemDataProvider;
use ApiPlatform\Core\DataProvider\ContextAwareCollectionDataProviderInterface;
use ApiPlatform\Core\DataProvider\DenormalizedIdentifiersAwareItemDataProviderInterface;
use ApiPlatform\Core\DataProvider\RestrictedDataProviderInterface;
use App\Entity\Item;
use App\Entity\ItemType;


class ItemDataProvider implements ContextAwareCollectionDataProviderInterface, DenormalizedIdentifiersAwareItemDataProviderInterface, RestrictedDataProviderInterface
{
    public function __construct(
        private CollectionDataProvider $doctrineCollectionDataProvider,
        private ItemDataProvider $doctrineItemDataProvider
    ) {
    }

    public function getCollection(
        string $resourceClass,
        string $operationName = null,
        array $context = []
    ): iterable {
        $items = $this->doctrineCollectionDataProvider->getCollection(
            $resourceClass,
            $operationName,
            $context
        );
        foreach ($items as $item) {
            $this->denormalizeItemTypes($item);
        }

        return $items;
    }

    public function getItem(
        string $resourceClass,
        $id,
        string $operationName = null,
        array $context = []
    ): ?object {
        $item = $this->doctrineItemDataProvider->getItem(
            $resourceClass,
            $id,
            $operationName,
            $context
        );
        if (!$item) {
            return null;
        }
        $this->denormalizeItemTypes($item);

        return $item;
    }

    public function supports(
        string $resourceClass,
        string $operationName = null,
        array $context = []
    ): bool {
        return Item::class === $resourceClass;
    }

    private function denormalizeItemTypes(Item $item): void
    {
        $item->initializeItemTypes();
        foreach ($item->getItemTypeIds() as $itemTypeId) {
            $item->addItemType(ItemType::from($itemTypeId));
        }
    }
}
```

Step 6 is to register the dependency injected services for both the `ItemDataPersister` and the `ItemDataProvider` classes
in `services.yaml`.

```yaml
# api/config/services.yaml
services:
    _defaults:
        bind:
            $decoratedDoctrineDataPersister: '@api_platform.doctrine.orm.data_persister'
            $doctrineCollectionDataProvider: '@api_platform.doctrine.orm.default.collection_data_provider'
            $doctrineItemDataProvider: '@api_platform.doctrine.orm.default.item_data_provider'
```

## Rationale

Why would one go to all this trouble instead of simply saving the codes in their own database tables?

Over the lifetime of the application, you are going to save billions of database read operations and data transfers.
The application's performance will also be improved because accessing codes is just an in-memory operation.

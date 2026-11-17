# Polymorphism with Discriminators

API Platform provides support for polymorphic resources using
[Symfony's Discriminator mapping](https://symfony.com/doc/current/serializer.html#deserializing-interfaces-and-abstract-classes)
in combination with
[Doctrine's inheritance mapping](https://www.doctrine-project.org/projects/doctrine-orm/en/current/reference/inheritance-mapping.html).
This allows you to represent a single API resource that can have multiple concrete implementations,
with each implementation exposing its own specific properties while sharing common properties from a
parent class.

## Overview

Polymorphism is useful when you need to:

- Return different types of objects from a single endpoint
- Have shared properties and behavior across multiple entity types
- Clearly communicate the structure of different entity variants through the OpenAPI schema
- Support object deserialization to the correct concrete type based on a discriminator property

## Setting Up Polymorphism

### Step 1: Create an Abstract Base Class

Define an abstract parent class that contains shared properties and the discriminator property.
Apply the `#[ApiResource]` attribute to make it an API resource:

```php
<?php
// api/src/Entity/Book.php
namespace App\Entity;

use ApiPlatform\Metadata\ApiResource;
use ApiPlatform\Metadata\Get;
use ApiPlatform\Metadata\GetCollection;
use Doctrine\ORM\Mapping as ORM;
use Symfony\Component\Serializer\Attribute\DiscriminatorMap;

#[ApiResource(operations: [
    new GetCollection(),
    new Get(),
])]
#[ORM\Entity]
#[ORM\InheritanceType('SINGLE_TABLE')]
#[ORM\DiscriminatorColumn(name: 'book_type', type: 'string')]
#[ORM\DiscriminatorMap([
    'fiction' => FictionBook::class,
    'technical' => TechnicalBook::class,
])]
#[DiscriminatorMap(typeProperty: 'bookType', mapping: [
    'fiction' => FictionBook::class,
    'technical' => TechnicalBook::class,
])]
abstract class Book
{
    #[ORM\Id]
    #[ORM\GeneratedValue]
    #[ORM\Column(type: 'integer')]
    private ?int $id = null;

    #[ORM\Column(type: 'string', length: 255)]
    private string $title;

    #[ORM\ManyToOne(targetEntity: Author::class)]
    #[ORM\JoinColumn(nullable: false)]
    private Author $author;

    #[ORM\Column(type: 'string', length: 20)]
    private string $isbn;

    public function __construct(string $title = '', ?Author $author = null, string $isbn = '')
    {
        $this->title = $title;
        $this->author = $author ?? new Author('Unknown');
        $this->isbn = $isbn;
    }

    public function getId(): ?int
    {
        return $this->id;
    }

    public function getTitle(): string
    {
        return $this->title;
    }

    public function setTitle(string $title): self
    {
        $this->title = $title;
        return $this;
    }

    public function getAuthor(): Author
    {
        return $this->author;
    }

    public function setAuthor(Author $author): self
    {
        $this->author = $author;
        return $this;
    }

    public function getIsbn(): string
    {
        return $this->isbn;
    }

    public function setIsbn(string $isbn): self
    {
        $this->isbn = $isbn;
        return $this;
    }

    abstract public function getBookType(): string;
}
```

### Key Attributes

- **`#[ORM\InheritanceType('SINGLE_TABLE')]`**: Tells Doctrine to use single-table inheritance
  strategy (all entities share one database table)
- **`#[ORM\DiscriminatorColumn(...)]`**: Specifies the database column that distinguishes between
  entity types
- **`#[ORM\DiscriminatorMap(...)]`**: Maps discriminator values to concrete entity classes
- **`#[DiscriminatorMap(...)]`**: Symfony Serializer attribute that maps JSON discriminator values
  to PHP classes

### Step 2: Create Concrete Child Classes

Create concrete entity classes that extend the abstract parent and define their own specific
properties:

```php
<?php
// api/src/Entity/FictionBook.php
namespace App\Entity;

use Doctrine\ORM\Mapping as ORM;

#[ORM\Entity]
class FictionBook extends Book
{
    public const string BOOK_TYPE = 'fiction';

    #[ORM\Column(type: 'string', length: 100, nullable: true)]
    private ?string $genre = null;

    #[ORM\Column(type: 'integer', nullable: true)]
    private ?int $pageCount = null;

    public function __construct(
        string $title = '',
        ?Author $author = null,
        string $isbn = '',
        ?string $genre = null,
        ?int $pageCount = null,
    ) {
        parent::__construct($title, $author, $isbn);
        $this->genre = $genre;
        $this->pageCount = $pageCount;
    }

    public function getBookType(): string
    {
        return self::BOOK_TYPE;
    }

    public function getGenre(): ?string
    {
        return $this->genre;
    }

    public function setGenre(?string $genre): self
    {
        $this->genre = $genre;
        return $this;
    }

    public function getPageCount(): ?int
    {
        return $this->pageCount;
    }

    public function setPageCount(?int $pageCount): self
    {
        $this->pageCount = $pageCount;
        return $this;
    }
}
```

```php
<?php
// api/src/Entity/TechnicalBook.php
namespace App\Entity;

use Doctrine\ORM\Mapping as ORM;

#[ORM\Entity]
class TechnicalBook extends Book
{
    public const string BOOK_TYPE = 'technical';

    #[ORM\Column(type: 'string', length: 100, nullable: true)]
    private ?string $programmingLanguage = null;

    #[ORM\Column(type: 'string', length: 50, nullable: true)]
    private ?string $difficultyLevel = null;

    #[ORM\Column(type: 'string', length: 255, nullable: true)]
    private ?string $topic = null;

    public function __construct(
        string $title = '',
        ?Author $author = null,
        string $isbn = '',
        ?string $programmingLanguage = null,
        ?string $difficultyLevel = null,
        ?string $topic = null,
    ) {
        parent::__construct($title, $author, $isbn);
        $this->programmingLanguage = $programmingLanguage;
        $this->difficultyLevel = $difficultyLevel;
        $this->topic = $topic;
    }

    public function getBookType(): string
    {
        return self::BOOK_TYPE;
    }

    public function getProgrammingLanguage(): ?string
    {
        return $this->programmingLanguage;
    }

    public function setProgrammingLanguage(?string $programmingLanguage): self
    {
        $this->programmingLanguage = $programmingLanguage;
        return $this;
    }

    public function getDifficultyLevel(): ?string
    {
        return $this->difficultyLevel;
    }

    public function setDifficultyLevel(?string $difficultyLevel): self
    {
        $this->difficultyLevel = $difficultyLevel;
        return $this;
    }

    public function getTopic(): ?string
    {
        return $this->topic;
    }

    public function setTopic(?string $topic): self
    {
        $this->topic = $topic;
        return $this;
    }
}
```

## API Response

When you fetch a collection of books from the API, each item will include the discriminator property
(`bookType`) that indicates its concrete type, along with all properties specific to that type:

```json
{
    "@context": "/api/contexts/Book",
    "id": "/api/books",
    "@type": "Collection",
    "totalItems": 2,
    "member": [
        {
            "@id": "/api/books/1",
            "@type": "Book",
            "genre": "Science Fiction",
            "pageCount": 450,
            "bookType": "fiction",
            "id": 1,
            "title": "The Great Novel",
            "author": "/api/authors/5",
            "isbn": "978-1234567890"
        },
        {
            "@id": "/api/books/2",
            "@type": "Book",
            "programmingLanguage": "PHP",
            "difficultyLevel": "Advanced",
            "topic": "Web Development",
            "bookType": "technical",
            "id": 2,
            "title": "Advanced PHP",
            "author": "/api/authors/7",
            "isbn": "978-0987654321"
        }
    ]
}
```

## OpenAPI Schema

API Platform automatically generates an OpenAPI schema with proper polymorphism support using
`oneOf` and `discriminator`:

```json
"Book.jsonld": {
    "allOf": [
        {
            "$ref": "#/components/schemas/HydraItemBaseSchema"
        },
        {
            "type": "object",
            "properties": {
                "id": {
                    "readOnly": true,
                    "type": "integer"
                },
                "title": {
                    "type": "string"
                },
                "author": {
                    "type": [
                        "string",
                        "null"
                    ],
                    "format": "iri-reference",
                    "example": "https://example.com/"
                },
                "isbn": {
                    "type": "string"
                },
                "bookType": {
                    "readOnly": true,
                    "type": "string"
                }
            },
            "oneOf": [
                {
                    "$ref": "#/components/schemas/FictionBook.jsonld"
                },
                {
                    "$ref": "#/components/schemas/TechnicalBook.jsonld"
                }
            ],
            "discriminator": {
                "propertyName": "bookType",
                "mapping": {
                    "fiction": "#/components/schemas/FictionBook.jsonld",
                    "technical": "#/components/schemas/TechnicalBook.jsonld"
                }
            }
        }
    ]
}
```

```json
"FictionBook.jsonld": {
    "allOf": [
        {
            "$ref": "#/components/schemas/Book.jsonld"
        },
        {
            "type": "object",
            "properties": {
                "genre": {
                    "type": [
                        "string",
                        "null"
                    ]
                },
                "pageCount": {
                    "type": [
                        "integer",
                        "null"
                    ]
                }
            }
        }
    ]
}
```

```json
"TechnicalBook.jsonld": {
    "allOf": [
        {
            "$ref": "#/components/schemas/Book.jsonld"
        },
        {
            "type": "object",
            "properties": {
                "programmingLanguage": {
                    "type": [
                        "string",
                        "null"
                    ]
                },
                "difficultyLevel": {
                    "type": [
                        "string",
                        "null"
                    ]
                },
                "topic": {
                    "type": [
                        "string",
                        "null"
                    ]
                }
            }
        }
    ]
}
```

## Deserialization

When creating or updating polymorphic resources, the discriminator property is required to determine
which concrete class to instantiate. For example, when POSTing a new book:

```json
{
    "title": "Design Patterns",
    "author": "/api/authors/3",
    "isbn": "978-0201633610",
    "bookType": "technical",
    "programmingLanguage": "Java",
    "difficultyLevel": "Intermediate",
    "topic": "Software Architecture"
}
```

The `bookType: "technical"` value tells the serializer to instantiate a `TechnicalBook` object and
populate its properties accordingly.

## Inheritance Mapping Strategies

Doctrine supports three inheritance strategies. API Platform works with all of them, though
`SINGLE_TABLE` is recommended for polymorphic APIs:

### SINGLE_TABLE (Recommended)

All entities share a single database table with a discriminator column:

```php
#[ORM\InheritanceType('SINGLE_TABLE')]
#[ORM\DiscriminatorColumn(name: 'type', type: 'string')]
#[ORM\DiscriminatorMap(['fiction' => FictionBook::class, 'technical' => TechnicalBook::class])]
abstract class Book
{
    // ...
}
```

**Advantages:**

- Single query to fetch all polymorphic objects
- Simple schema
- Best performance

**Disadvantages:**

- Child-specific columns are nullable
- All child properties share the same table

### JOINED

Each entity class has its own table, with a foreign key relationship to the parent:

```php
#[ORM\InheritanceType('JOINED')]
#[ORM\DiscriminatorColumn(name: 'type', type: 'string')]
#[ORM\DiscriminatorMap(['fiction' => FictionBook::class, 'technical' => TechnicalBook::class])]
abstract class Book
{
    // ...
}
```

**Advantages:**

- No null columns in child tables
- Better data normalization

**Disadvantages:**

- Requires JOIN operations
- More complex queries

### TABLE_PER_CLASS

Each entity class has its own complete table with all properties:

```php
#[ORM\InheritanceType('TABLE_PER_CLASS')]
abstract class Book
{
    // ...
}
```

> Note: This strategy doesn't require a discriminator column as Doctrine can infer the type from the
> table.

## Troubleshooting

### Properties from Child Classes Not Appearing in Response

Ensure that:

1. Child class properties are marked with `#[ORM\Column]`
2. The `#[DiscriminatorMap]` attribute is present on the parent class
3. Child class properties have public getters (or are public properties)

### Serialization Errors

If you encounter serialization errors:

1. Verify the `#[DiscriminatorMap]` mapping is correct
2. Check that the discriminator property name matches between Doctrine and Serializer attributes
3. Ensure child classes properly call `parent::__construct()` if needed

### OpenAPI Schema Not Showing Child Properties

This might indicate:

1. The discriminator attribute is misconfigured
2. Child class properties lack proper type hints

## See Also

- [Symfony Serializer Discriminator Documentation](https://symfony.com/doc/current/serializer.html#deserializing-interfaces-and-abstract-classes)
- [Doctrine Inheritance Mapping](https://www.doctrine-project.org/projects/doctrine-orm/en/current/reference/inheritance-mapping.html)
- [JSON-LD and Polymorphism](https://json-ld.org/)
- [OpenAPI 3.0 Discriminator Object](https://spec.openapis.org/oas/v3.0.0#discriminator-object)

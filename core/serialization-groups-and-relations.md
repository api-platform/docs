# Serialization Groups and Relations

API Platform Core allows to choose which attributes of the resource are exposed during the normalization (read) and denormalization
(write) process. It relies on the [serialization (and deserialization) groups](https://symfony.com/doc/current/components/serializer.html#attributes-groups)
feature of the Symfony Serializer component.

## Configuration

The Symfony Serializer component allows to specify the definition of serialization using XML, YAML, or annotations. As annotations
are really easy to understand, we will use them in this documentation.

However, if you don't use the standard edition of API Platform, don't forget to enable annotation support in the serializer
configuration:

```yaml
# app/config/config.yml

framework:
    serializer: { enable_annotations: true }
```

## Using Serialization Groups

Specifying to the API system the groups to use is really simple:

```php
// src/AppBundle/Entity/Book.php

namespace AppBundle\Entity;

use ApiPlatform\Core\Annotation\ApiResource;
use Symfony\Component\Serializer\Annotation\Groups;

/**
 * @ApiResource(attributes={
 *     "normalization_context"={"groups"={"read"}},
 *     "denormalization_context"={"groups"={"write"}}
 * })
 */
class Book
{
    /**
     * @Groups({"read", "write"})
     */
    private $name;

    /**
     * @Groups({"write"})
     */
    private $author;

    // ...
}
```

With the config of the previous example, the `name` property will be accessible in read and write, but the `author` property
will be write only, therefore the `author` property will never be included in documents returned by the API.

The value of the `normalization_context` is passed to the Symfony Serializer during the normalization process. In the same
way, `denormalization_context` is used for denormalization.
You can configure groups as well as any Symfony Serializer option configurable through the context argument (e.g. the `enable_max_depth`
key when using [the `@MaxDepth` annotation](https://github.com/symfony/symfony/issues/17113)).

Built-in actions and the Hydra documentation generator will leverage the specified serialization and deserialization groups
to give access only to exposed properties and to guess if they are readable and/or writable.

## Using Different Serialization Groups per Operation

It is possible to specify normalization and denormalization contexts (as well as any other attribute) on a per operation
basis. API Platform Core will always use the most specific definition. For instance if normalization groups are set both
at the resource level and at the operation level, the configuration set at the operation level will be used and the resource
level ignored.

In the following example we use different serialization groups for the `GET` and `PUT` operations:

```php
// src/AppBundle/Entity/Book.php

namespace AppBundle\Entity;

use ApiPlatform\Core\Annotation\ApiResource;
use Symfony\Component\Serializer\Annotation\Groups;

/**
 * @ApiResource(
 *     attributes={"normalization_context"={"groups"={"get"}}},
 *     itemOperations={
 *          "get"={"method"="GET"},
 *          "put"={"method"="PUT", "normalization_context"={"groups"={"put"}}}
 *     }
 * )
 */
class Book
{
    /**
     * @Groups({"get", "put"})
     */
    private $name;

    /**
     * @Groups({"get"})
     */
    private $author;

    // ...
}
```

`name` and `author` properties will be included in the document generated during a `GET` operation because the configuration
defined at the resource level is inherited. However the document generated when a `PUT` request will be received will only
include the `name` property because of the specific configuration for this operation.

Refer to the documentation of [operations](operations.md) to learn more about the concept of operations.

## Embedding Relations

By default, the serializer provided with API Platform Core represents relations between objects by [dereferenceables IRIs](https://en.wikipedia.org/wiki/Internationalized_Resource_Identifier).
They allow to retrieve details of related objects by issuing an extra HTTP request.

In the following JSON document, the relation from a book to an author is represented by an URI:

```json
{
  "@context": "/contexts/Book",
  "@id": "/books/62",
  "@type": "Book",
  "name": "My awesome book",
  "author": "/people/59"
}
```

### Normalization

To improve the application's performance, it is sometimes necessary to avoid issuing extra HTTP requests. It is possible
to embed related objects (or only some of their properties) directly in the parent response trough serialization groups.
By using the following serialization groups annotations (`@Groups`), a JSON representation of the author is embedded in
the book response:

```php
// src/AppBundle/Entity/Book.php

namespace AppBundle\Entity;

use ApiPlatform\Core\Annotation\ApiResource;
use Symfony\Component\Serializer\Annotation\Groups;

/**
 * @ApiResource(attributes={"normalization_context"={"groups"={"book"}}})
 */
class Book
{
    /**
     * @Groups({"book"})
     */
    private $name;
    
    /**
     * @Groups({"book"})
     */
    private $author;

    // ...
}
```

```php
// src/AppBundle/Entity/Person.php

namespace AppBundle\Entity;

use ApiPlatform\Core\Annotation\ApiResource;
use Symfony\Component\Serializer\Annotation\Groups;

/**
 * @ApiResource
 */
class Person
{
    /**
     * ...
     * @Groups({"book"})
     */
    public $name;

    // ...
}
```

The generated JSON with previous settings will be like the following:

```json
{
  "@context": "/contexts/Book",
  "@id": "/books/62",
  "@type": "Book",
  "name": "My awesome book",
  "author": {
    "@id": "/people/59",
    "@type": "Person",
    "name": "Kévin Dunglas"
  }
}
```

In order to optimize such embedded relations, the default Doctrine data provider will automatically join entities on relations
marked as [`EAGER`](http://doctrine-orm.readthedocs.io/projects/doctrine-orm/en/latest/reference/annotations-reference.html#manytoone)
avoiding extra queries to be executed when serializing the sub-objects.

### Denormalization

It is also possible to embed a relation in `PUT` and `POST` requests. To enable that feature, the serialization groups must be
set the same way as normalization and the configuration should be like this:

```php
// src/AppBundle/Entity/Book.php

namespace AppBundle\Entity;

use ApiPlatform\Core\Annotation\ApiResource;

/**
 * @ApiResource(attributes={"denormalization_context"={"groups"={"book"}}})
 */
class Book
{
    // ...
}
```

The following rules apply when denormalizating embedded relations:

* If a `@id` key is present in the embedded resource, the object corresponding to the given URI will be retrieved trough
the data provider and any changes in the embedded relation will be applied to that object.
* If no `@id` key exists, a new object will be created containing data provided in the embedded JSON document.

You can create as relation embedding levels as you want.

## Name Conversion

The Serializer Component provides a handy way to map PHP field names to serialized names. See the related [Symfony documentation](http://symfony.com/doc/master/components/serializer.html#converting-property-names-when-serializing-and-deserializing).

To use this feature, declare a new service with id `api.name_converter`. For example, you can convert `CamelCase` to 
`snake_case` with the following configuration:

```yaml
# app/config/services.yml

services:
    api.name_converter:
        class: Symfony\Component\Serializer\NameConverter\CamelCaseToSnakeCaseNameConverter
        public: false
```

## Entity Identifier Case

API Platform is able to guess the entity identifier using [Doctrine metadata](http://doctrine-orm.readthedocs.org/en/latest/reference/basic-mapping.html#identifiers-primary-keys).
It also supports composite identifiers.

If Doctrine ORM is not used, the identifier must be marked explicitly using the `identifier` attribute of the `ApiPlatform\Core\Annotation\ApiProperty`
annotation.

Most of the time, the property identifying the entity is not included in the returned document but is included as a part
of the URI contained in the `@id` field. So in the `/apidoc` endpoint the identifier will not appear in the properties list.

However, when using composite identifier, properties composing the identifier are included in the API response and in the
documentation.

### Writable Entity Identifier

In some cases, you will want to set the identifier of a resource from the client (like a slug for example).
In this case the identifier property must become a writable class property in the `/apidoc` endpoint.

To do this you simply have to:

* Create a setter for identifier in the entity.
* Add the denormalization group to the property if you use a specific denormalization group.

## Embedding the Context

By default, the generated [JSON-LD context](https://www.w3.org/TR/json-ld/#the-context) (`@context`) is only reference by
an IRI. A client supporting JSON-LD must send a second HTTP request to retrieve it:

```json
{
  "@context": "/contexts/Book",
  "@id": "/books/62",
  "@type": "Book",
  "name": "My awesome book",
  "author": "/people/59"
}
```

You can configure API Platform Core to embed the JSON-LD context in the root document like the following:

```json
{
  "@context": {
    "@vocab": "http://localhost:8000/apidoc#",
    "hydra": "http://www.w3.org/ns/hydra/core#",
    "name": "http://schema.org/name",
    "author": "http://schema.org/authhor"
  },
  "@id": "/books/62",
  "@type": "Book",
  "name": "My awesome book",
  "author": "/people/59"
}
```

To do so, use the following configuration:

```php
// src/AppBundle/Entity/Book.php

namespace AppBundle\Entity;

use ApiPlatform\Core\Annotation\ApiResource;

/**
 * @ApiResource(attributes={"jsonld_embed_context"=true})
 */
class Book
{
    // ...
}
```

Previous chapter: [Filters](filters.md)<br>
Next chapter: [Resources path generators](resource-path-generator.md)

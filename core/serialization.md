# The Serialization Process

## Overall Process

API Platform embraces and extends the Symfony Serializer Component to transform PHP entities in (hypermedia) API responses.

The main serialization process has two stages:

![Serializer workflow](images/SerializerWorkflow.png)

> As you can see in the picture above, an array is used as a man in the middle. This way, Encoders will only deal with turning specific formats into arrays and vice versa. The same way, Normalizers will deal with turning specific objects into arrays and vice versa.
-- [The Symfony documentation](https://symfony.com/doc/current/components/serializer.html)

Unlike Symfony itself, API Platform leverages custom normalizers, its router and the [data provider](data-providers.md) system to do an advanced tranformation. Metadata are added to the generated document including links, type information, pagination data or available filters.

The API Platform Serializer is extendable, you can register custom normalizers and encoders to support other formats. You can also decorate existing normalizers to customize their behaviors.

## Available Serializers

* [JSON-LD](https://json-ld.org) serializer
`api_platform.jsonld.normalizer.item`

JSON-LD, or JavaScript Object Notation for Linked Data, is a method of encoding Linked Data using JSON. It is a World Wide Web Consortium Recommendation.

* [HAL](https://en.wikipedia.org/wiki/Hypertext_Application_Language) serializer
`api_platform.hal.normalizer.item`

* JSON, XML, CSV, YAML serializer (using the Symfony serializer)
`api_platform.serializer.normalizer.item`

## The Serialization Context, Groups and Relations

API Platform allows to specify the `$context` parameter used by the Symfony Serializer. This context has a handy
`groups` key allowing to choose which attributes of the resource are exposed during the normalization (read) and denormalization
(write) process.
It relies on the [serialization (and deserialization) groups](https://symfony.com/doc/current/components/serializer.html#attributes-groups)
feature of the Symfony Serializer component.

In addition to groups, you can use any option supported by the Symfony Serializer such as [`enable_max_depth`](https://symfony.com/doc/current/components/serializer.html#handling-serialization-depth)
to limit the serialization depth.

### Configuration

Just like other Symfony and API Platform components, the Serializer can be configured using annotations, XML and YAML.
As annotations are easy to understand and allow to group code and configuration, we will use them in the following examples.

However, if you don't use the official distribution of API Platform, don't forget to enable annotation support in the serializer
configuration:

```yaml
# api/config/packages/api_platform.yaml
framework:
    serializer: { enable_annotations: true }
```

If you use [Symfony Flex](https://github.com/symfony/flex), just execute `composer req doctrine/annotations` and you are
all set!

## Using Serialization Groups

It is really simple to specify what groups to use in the API system:

```php
<?php
// api/src/Entity/Book.php

namespace App\Entity;

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
     * @Groups("write")
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
key when using [the `@MaxDepth` annotation](https://symfony.com/doc/current/components/serializer.html#handling-serialization-depth)).

Built-in actions and the Hydra documentation generator will leverage the specified serialization and deserialization groups
to give access only to exposed properties and to guess if they are readable and/or writable.

## Using Different Serialization Groups per Operation

It is possible to specify normalization and denormalization contexts (as well as any other attribute) on a per operation
basis. API Platform will always use the most specific definition. For instance if normalization groups are set both
at the resource level and at the operation level, the configuration set at the operation level will be used and the resource
level ignored.

In the following example we use different serialization groups for the `GET` and `PUT` operations:

```php
<?php
// api/src/Entity/Book.php

namespace App\Entity;

use ApiPlatform\Core\Annotation\ApiResource;
use Symfony\Component\Serializer\Annotation\Groups;

/**
 * @ApiResource(
 *     attributes={
 *         "normalization_context"={"groups"={"get"}}
 *     },
 *     itemOperations={
 *          "get",
 *          "put"={"normalization_context"={"groups"={"put"}}}
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
     * @Groups("get")
     */
    private $author;

    // ...
}
```

`name` and `author` properties will be included in the document generated during a `GET` operation because the configuration
defined at the resource level is inherited. However the document generated when a `PUT` request will be received will only
include the `name` property because of the specific configuration for this operation.

Refer to the documentation of [operations](operations.md) to learn more about the concept of operations.

### Embedding Relations

By default, the serializer provided with API Platform represents relations between objects by [dereferenceables IRIs](https://en.wikipedia.org/wiki/Internationalized_Resource_Identifier).
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
to embed related objects (or only some of their properties) directly in the parent response through serialization groups.
By using the following serialization groups annotations (`@Groups`), a JSON representation of the author is embedded in
the book response:

```php
<?php
// api/src/Entity/Book.php

namespace App\Entity;

use ApiPlatform\Core\Annotation\ApiResource;
use Symfony\Component\Serializer\Annotation\Groups;

/**
 * @ApiResource(attributes={
 *     "normalization_context"={"groups"={"book"}}
 * })
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
<?php
// api/src/Entity/Person.php

namespace App\Entity;

use ApiPlatform\Core\Annotation\ApiResource;
use Symfony\Component\Serializer\Annotation\Groups;

/**
 * @ApiResource
 */
class Person
{
    /**
     * ...
     * @Groups("book")
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
<?php
// api/src/Entity/Book.php

namespace App\Entity;

use ApiPlatform\Core\Annotation\ApiResource;

/**
 * @ApiResource(attributes={
 *     "denormalization_context"={"groups"={"book"}}
 * })
 */
class Book
{
    // ...
}
```

The following rules apply when denormalizating embedded relations:

* If a `@id` key is present in the embedded resource, the object corresponding to the given URI will be retrieved through
the data provider and any changes in the embedded relation will be applied to that object.
* If no `@id` key exists, a new object will be created containing data provided in the embedded JSON document.

You can create as relation embedding levels as you want.

## Changing the Serialization Context Dynamically

Let's imagine a resource where most fields can be managed by any user, but some can be managed by admin users only:

```php
<?php
// api/src/Entity/Book.php

namespace App\Entity;

use ApiPlatform\Core\Annotation\ApiResource;
use Symfony\Component\Serializer\Annotation\Groups;

/**
 * @ApiResource(attributes={
 *     "normalization_context"={"groups"={"book_output"}},
 *     "denormalization_context"={"groups"={"book_input"}}
 * })
 */
class Book
{
    // ...

    /**
     * This field can be managed by an admin only
     *
     * @var bool
     *
     * @Groups({"book_output", "admin_input"})
     */
    private $active = false;

    /**
     * This field can be managed by any user
     *
     * @var string
     *
     * @Groups({"book_output", "book_input"})
     */
    private $name;

    // ...
}
```

All entry points are the same for all users, so we should find a way to detect if authenticated user is admin, and if so
dynamically add `admin_input` to deserialization groups.

API Platform implements a `ContextBuilder`, which prepares the context for serialization & deserialization. Let's
[decorate this service](http://symfony.com/doc/current/service_container/service_decoration.html) to override the
`createFromRequest` method:

```yaml
# api/config/services.yaml
services:
    # ...
    'App\Serializer\BookContextBuilder':
        decorates: 'api_platform.serializer.context_builder'
        arguments: [ '@App\Serializer\BookContextBuilder.inner' ]
        autoconfigure: false
```

```php
<?php
// api/src/Serializer/BookContextBuilder.php

namespace App\Serializer;

use ApiPlatform\Core\Serializer\SerializerContextBuilderInterface;
use Symfony\Component\HttpFoundation\Request;
use Symfony\Component\Security\Core\Authorization\AuthorizationCheckerInterface;
use App\Entity\Book;

final class BookContextBuilder implements SerializerContextBuilderInterface
{
    private $decorated;
    private $authorizationChecker;

    public function __construct(SerializerContextBuilderInterface $decorated, AuthorizationCheckerInterface $authorizationChecker)
    {
        $this->decorated = $decorated;
        $this->authorizationChecker = $authorizationChecker;
    }

    public function createFromRequest(Request $request, bool $normalization, ?array $extractedAttributes = null): array
    {
        $context = $this->decorated->createFromRequest($request, $normalization, $extractedAttributes);
        $resourceClass = $context['resource_class'] ?? null;
        
        if ($resourceClass === Book::class && isset($context['groups']) && $this->authorizationChecker->isGranted('ROLE_ADMIN') && false === $normalization) {
            $context['groups'][] = 'admin_input';
        }

        return $context;
    }
}
```

If the user has `ROLE_ADMIN` permission and the subject is an instance of Book, `admin_input` group will be dynamically added to the denormalization context.
The variable `$normalization` lets you check whether the context is for normalization (if true) or denormalization.

## Changing the Serialization Context on a Per Item Basis

The example above shows how you can modify the normalization/denormalization context based on the current user permissions for all the books that are being normalized/denormalized. Sometimes, however, the permissions vary depending on what book is being processed. Think of ACL's: User A may retrieve Book A but not Book B. In that case, we have to leverage the power of the Symfony Serializer and register our own normalizer that adds the group on every single item (priority `64` is just an example, make sure your normalizer gets loaded first):

```yaml
# api/config/services.yaml
services:
    'App\Serializer\BookAttributeNormalizer':
        arguments: [ '@security.token_storage' ]
        tags:
            - { name: 'serializer.normalizer', priority: 64 }
```

The Normalizer class is a bit harder to understand because it has to make sure there's no recursion. To do so, it has to be aware of the Serializer instance itself to pass on the object to normalize to the other normalizers once it added the groups. Here's how this could look like:

```php
<?php
// api/src/Serializer/BookAttributeNormalizer.php

namespace App\Serializer;

class BookAttributeNormalizer implements ContextAwareNormalizerInterface, SerializerAwareInterface
{
    use SerializerAwareTrait;

    private const BOOK_ATTRIBUTE_NORMALIZER_ALREADY_CALLED = 'BOOK_ATTRIBUTE_NORMALIZER_ALREADY_CALLED';

    private $tokenStorage;

    public function __construct(TokenStorageInterface $tokenStorage)
    {
        $this->tokenStorage = $tokenStorage;
    }

    public function normalize($object, $format = null, array $context = [])
    {
        if ($this->userHasPermissionsForBook($object)) {
            $context['groups'][] = 'can_retrieve_book';
        }

        return $this->passOn($object, $format, $context);
    }
    
    public function supportsNormalization($data, $format = null, array $context = [])
    {
        // Make sure we're not called twice
        if (isset($context[self::BOOK_ATTRIBUTE_NORMALIZER_ALREADY_CALLED])) {
            return false;
        }

        return $data instanceof Book;
    }
    
    private function userHasPermissionsForBook($object): bool
    {
        // Get permissions from user in $this->tokenStorage
        // for the current $object (book) and
        // return true or false
    }

    private function passOn($object, $format = null, array $context = [])
    {
        if (!$this->serializer instanceof NormalizerInterface) {
            throw new LogicException(sprintf('Cannot normalize object "%s" because the injected serializer is not a normalizer', $object));
        }

        $context[self::BOOK_ATTRIBUTE_NORMALIZER_ALREADY_CALLED] = true;

        return $this->serializer->normalize($object, $format, $context);
    }
}
```

This will add the serialization group `can_retrieve_book` only if the currently logged in user has access to the given book instance.

Note: In this example, we use the `TokenStorageInterface` to check for access. Don't forget that Symfony provides many useful other services that might be better suited depending on your use case such as the [`AuthorizationChecker`](https://symfony.com/doc/current/components/security/authorization.html#authorization-checker)

## Name Conversion

The Serializer Component provides a handy way to map PHP field names to serialized names. See the related [Symfony documentation](http://symfony.com/doc/master/components/serializer.html#converting-property-names-when-serializing-and-deserializing).

To use this feature, declare a new service with id `app.name_converter`. For example, you can convert `CamelCase` to
`snake_case` with the following configuration:

```yaml
# api/config/services.yaml
services:
    'Symfony\Component\Serializer\NameConverter\CamelCaseToSnakeCaseNameConverter': ~
```

```yaml
# api/config/packages/api_platform.yaml
api_platform:
    name_converter: 'Symfony\Component\Serializer\NameConverter\CamelCaseToSnakeCaseNameConverter'
```

## Decorating a Serializer and Add Extra Data

In the following example, we will see how we add extra informations to the output.
Here is how we add the date on each request in `GET`:

```yaml
# api/config/services.yaml
services:
    'App\Serializer\ApiNormalizer':
        decorates: 'api_platform.jsonld.normalizer.item'
        arguments: [ '@App\Serializer\ApiNormalizer.inner' ]
```

```php
<?php
// api/src/Serializer/ApiNormalizer

namespace App\Serializer;

use Symfony\Component\Serializer\Normalizer\DenormalizerInterface;
use Symfony\Component\Serializer\Normalizer\NormalizerInterface;
use Symfony\Component\Serializer\SerializerAwareInterface;
use Symfony\Component\Serializer\SerializerInterface;

final class ApiNormalizer implements NormalizerInterface, DenormalizerInterface, SerializerAwareInterface
{
    private $decorated;

    public function __construct(NormalizerInterface $decorated)
    {
        if (!$decorated instanceof DenormalizerInterface) {
            throw new \InvalidArgumentException(sprintf('The decorated normalizer must implement the %s.', DenormalizerInterface::class));
        }

        $this->decorated = $decorated;
    }

    public function supportsNormalization($data, $format = null)
    {
        return $this->decorated->supportsNormalization($data, $format);
    }

    public function normalize($object, $format = null, array $context = [])
    {
        $data = $this->decorated->normalize($object, $format, $context);
        if (is_array($data)) {
            $data['date'] = date(\DateTime::RFC3339);
        }

        return $data;
    }

    public function supportsDenormalization($data, $type, $format = null)
    {
        return $this->decorated->supportsDenormalization($data, $type, $format);
    }

    public function denormalize($data, $class, $format = null, array $context = [])
    {
        return $this->decorated->denormalize($data, $class, $format, $context);
    }
    
    public function setSerializer(SerializerInterface $serializer)
    {
        if($this->decorated instanceof SerializerAwareInterface) {
            $this->decorated->setSerializer($serializer);
        }
    }
}
```

## Entity Identifier Case

API Platform is able to guess the entity identifier using [Doctrine metadata](http://doctrine-orm.readthedocs.org/en/latest/reference/basic-mapping.html#identifiers-primary-keys).
It also supports composite identifiers.

If Doctrine ORM is not used, the identifier must be marked explicitly using the `identifier` attribute of the `ApiPlatform\Core\Annotation\ApiProperty`
annotation.

In some cases, you will want to set the identifier of a resource from the client (e.g. a client-side generated UUID, or a slug).
In this case the identifier property must become a writable class property.

To do this you simply have to:

* Create a setter for the identifier of the entity or make it a `public` property
* Add the denormalization group to the property if you use a specific denormalization group
* If you use Doctrine ORM, be sure to **not** mark this property with [the `@GeneratedValue` annotation](http://docs.doctrine-project.org/projects/doctrine-orm/en/latest/reference/basic-mapping.html#identifier-generation-strategies)
  or use the `NONE` value

## Embedding the JSON-LD Context

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

You can configure API Platform to embed the JSON-LD context in the root document like the following:

```json
{
  "@context": {
    "@vocab": "http://localhost:8000/apidoc#",
    "hydra": "http://www.w3.org/ns/hydra/core#",
    "name": "http://schema.org/name",
    "author": "http://schema.org/author"
  },
  "@id": "/books/62",
  "@type": "Book",
  "name": "My awesome book",
  "author": "/people/59"
}
```

To do so, use the following configuration:

```php
<?php
// api/src/Entity/Book.php

namespace App\Entity;

use ApiPlatform\Core\Annotation\ApiResource;

/**
 * @ApiResource(
 *     attributes={"normalization_context"={"jsonld_embed_context"=true}
 * })
 */
class Book
{
    // ...
}
```

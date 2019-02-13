# The Serialization Process

## Overall Process

API Platform embraces and extends the Symfony Serializer Component to transform PHP entities in (hypermedia) API responses.

The main serialization process has two stages:

![Serializer workflow](images/SerializerWorkflow.png)

> As you can see in the picture above, an array is used as a man-in-the-middle. This way, Encoders will only deal with turning specific formats into arrays and vice versa. The same way, Normalizers will deal with turning specific objects into arrays and vice versa.
-- [The Symfony documentation](https://symfony.com/doc/current/components/serializer.html)

Unlike Symfony itself, API Platform leverages custom normalizers, its router and the [data provider](data-providers.md) system to do an advanced transformation. Metadata are added to the generated document including links, type information, pagination data or available filters.

The API Platform Serializer is extendable. You can register custom normalizers and encoders in order to support other formats. You can also decorate existing normalizers to customize their behaviors.

## Available Serializers

* [JSON-LD](https://json-ld.org) serializer
`api_platform.jsonld.normalizer.item`

JSON-LD, or JavaScript Object Notation for Linked Data, is a method of encoding Linked Data using JSON. It is a World Wide Web Consortium Recommendation.

* [HAL](https://en.wikipedia.org/wiki/Hypertext_Application_Language) serializer
`api_platform.hal.normalizer.item`

* JSON, XML, CSV, YAML serializer (using the Symfony serializer)
`api_platform.serializer.normalizer.item`

## The Serialization Context, Groups and Relations

API Platform allows you to specify the `$context` variable used by the Symfony Serializer. This variable is an associative array that has a handy `groups` key allowing you to choose which attributes of the resource are exposed during the normalization (read) and denormalization (write) processes.
It relies on the [serialization (and deserialization) groups](https://symfony.com/doc/current/components/serializer.html#attributes-groups)
feature of the Symfony Serializer component.

In addition to groups, you can use any option supported by the Symfony Serializer. For example, you can use [`enable_max_depth`](https://symfony.com/doc/current/components/serializer.html#handling-serialization-depth)
to limit the serialization depth.

### Configuration

Just like other Symfony and API Platform components, the Serializer component can be configured using annotations, XML or YAML.
Since annotations are easy to understand, we will use them in the following examples.

Note: if you aren't using the official distribution of API Platform, you will need to enable annotation support in the serializer
configuration:

```yaml
# api/config/packages/api_platform.yaml
framework:
    serializer: { enable_annotations: true }
```

If you use [Symfony Flex](https://github.com/symfony/flex), just execute `composer req doctrine/annotations` and you are
all set!

If you want to use YAML or XML, please add the mapping path in the serializer configuration:

```yaml
# api/config/packages/api_platform.yaml
framework:
    serializer:
        mapping:
            paths: ['%kernel.project_dir%/config/serialization']
```

## Using Serialization Groups

It is simple to specify what groups to use in the API system:

1. Add the `normalizationContext` and `denormalizationContext` annotation properties to the `@ApiResource` annotation, and specify which groups to use. Here you see that we add `read` and `write`, respectively. You can use any group names you wish.
2. Apply the `@Groups` annotation to properties in the object.

```php
<?php
// api/src/Entity/Book.php

namespace App\Entity;

use ApiPlatform\Core\Annotation\ApiResource;
use Symfony\Component\Serializer\Annotation\Groups;

/**
 * @ApiResource(
 *     normalizationContext={"groups"={"read"}},
 *     denormalizationContext={"groups"={"write"}}
 * )
 */
class Book
{
    /**
     * @Groups({"read", "write"})
     */
    public $name;

    /**
     * @Groups("write")
     */
    public $author;

    // ...
}
```

Alternatively, you can use the more verbose syntax:

```php
<?php
// ...

/**
 * @ApiResource(attributes={
 *     "normalization_context"={"groups"={"read"}},
 *     "denormalization_context"={"groups"={"write"}}
 * })
 */
```

You can also use the YAML configuration format:

```yaml
# api/config/api_platform/resources.yaml
App\Entity\Book:
    attributes:
        normalization_context:
            groups: ['read']
        denormalization_context:
            groups: ['write']
```

```yaml
# api/config/serialization/Book.yaml
App\Entity\Book:
    attributes:
        name:
            groups: ['read', 'write']
        author:
            groups: ['write']
```

In the previous example, the `name` property will be visible when reading (`GET`) the object, and it will also be available
to write (`PUT/POST`). The `author` property will be write-only; it will not be visible when serialized responses are 
returned by the API.

Internally, API Platform passes the value of the `normalization_context` as the 3rd argument of [the `Serializer::serialize()` method](https://api.symfony.com/master/Symfony/Component/Serializer/SerializerInterface.html#method_serialize) during the normalization
process; `denormalization_context` is passed as the 4th argument of [the `Serializer::deserialize()` method](https://api.symfony.com/master/Symfony/Component/Serializer/SerializerInterface.html#method_deserialize) during denormalization (writing).

To configure the serialization groups of classes's properties, you must use directly [the Symfony Serializer's configuration files or annotations](https://symfony.com/doc/current/components/serializer.html#attributes-groups).


In addition to the `groups` key, you can configure any Symfony Serializer option through the `$context` parameter
(e.g. the `enable_max_depth`key when using [the `@MaxDepth` annotation](https://symfony.com/doc/current/components/serializer.html#handling-serialization-depth)).

Any serialization and deserialization groups that you specify will also be leveraged by the built-in actions and the Hydra
documentation generator.

## Using Serialization Groups per Operation

It is possible to specify normalization and denormalization contexts (as well as any other attribute) on a per-operation
basis. API Platform will always use the most specific definition. For instance, if normalization groups are set both
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
 *     normalizationContext={"groups"={"get"}},
 *     itemOperations={
 *         "get",
 *         "put"={
 *             "normalization_context"={"groups"={"put"}}
 *         }
 *     }
 * )
 */
class Book
{
    /**
     * @Groups({"get", "put"})
     */
    public $name;

    /**
     * @Groups("get")
     */
    public $author;

    // ...
}
```

The `name` and `author` properties will be included in the document generated during a `GET` operation because the configuration
defined at the resource level is inherited. However the document generated when a `PUT` request will be received will only
include the `name` property because of the specific configuration for this operation.

Refer to the [operations](operations.md) documentation to learn more.

### Embedding Relations

By default, the serializer provided with API Platform represents relations between objects using [dereferenceables IRIs](https://en.wikipedia.org/wiki/Internationalized_Resource_Identifier).
They allow you to retrieve details for related objects by issuing extra HTTP requests.

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

However, for performance reasons, it is sometimes preferable to avoid forcing the client to issue extra HTTP requests. 
It is possible to embed related objects (in their entirety, or only some of their properties) directly in the parent 
response through the use of serialization groups. By using the following serialization groups annotations (`@Groups`), 
a JSON representation of the author is embedded in the book response:

```php
<?php
// api/src/Entity/Book.php

namespace App\Entity;

use ApiPlatform\Core\Annotation\ApiResource;
use Symfony\Component\Serializer\Annotation\Groups;

/**
 * @ApiResource(normalizationContext={"groups"={"book"}})
 */
class Book
{
    /**
     * @Groups({"book"})
     */
    public $name;

    /**
     * @Groups({"book"})
     */
    public $author;

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

The generated JSON using previous settings is below:

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
marked as [`EAGER`](http://doctrine-orm.readthedocs.io/projects/doctrine-orm/en/latest/reference/annotations-reference.html#manytoone).
This avoids the need for extra queries to be executed when serializing the related objects.

Instead of embedding relation in the main HTTP response, you may want [to "push" them to the client using HTTP/2 server push](push-relations.md).

### Denormalization

It is also possible to embed a relation in `PUT` and `POST` requests. To enable that feature, set the serialization groups
the same way as normalization. For example:

```php
<?php
// api/src/Entity/Book.php

namespace App\Entity;

use ApiPlatform\Core\Annotation\ApiResource;

/**
 * @ApiResource(denormalizationContext={"groups"={"book"}})
 */
class Book
{
    // ...
}
```

The following rules apply when denormalizing embedded relations:

* If an `@id` key is present in the embedded resource, then the object corresponding to the given URI will be retrieved through
the data provider. Any changes in the embedded relation will also be applied to that object.
* If no `@id` key exists, a new object will be created containing data provided in the embedded JSON document.

You can specify as many embedded relation levels as you want.

## Changing the Serialization Context Dynamically

Let's imagine a resource where most fields can be managed by any user, but some can be managed only by admin users:

```php
<?php
// api/src/Entity/Book.php

namespace App\Entity;

use ApiPlatform\Core\Annotation\ApiResource;
use Symfony\Component\Serializer\Annotation\Groups;

/**
 * @ApiResource(
 *     normalizationContext={"groups"={"book:output"}},
 *     denormalizationContext={"groups"={"book:input"}}
 * )
 */
class Book
{
    // ...

    /**
     * This field can be managed only by an admin
     *
     * @var bool
     *
     * @Groups({"book:output", "admin:input"})
     */
    public $active = false;

    /**
     * This field can be managed by any user
     *
     * @var string
     *
     * @Groups({"book:output", "book:input"})
     */
    public $name;

    // ...
}
```

All entry points are the same for all users, so we should find a way to detect if authenticated user is an admin, and if so
dynamically add `admin:input` value to deserialization groups in the `$context` array.

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
            $context['groups'][] = 'admin:input';
        }

        return $context;
    }
}
```

If the user has the `ROLE_ADMIN` permission and the subject is an instance of Book, `admin_input` group will be dynamically added to the 
denormalization context. The `$normalization` variable lets you check whether the context is for normalization (if `TRUE`) or denormalization 
(`FALSE`).

## Changing the Serialization Context on a Per-item Basis

The example above demonstrates how you can modify the normalization/denormalization context based on the current user 
permissions for all books. Sometimes, however, the permissions vary depending on what book is being processed.

Think of ACL's: User "A" may retrieve Book "A" but not Book "B". In this case, we need to leverage the power of the 
Symfony Serializer and register our own normalizer that adds the group on every single item (note: priority `64` is 
an example; it is always important to make sure your normalizer gets loaded first, so set the priority to whatever value
is appropriate for your application; higher values are loaded earlier):

```yaml
# api/config/services.yaml
services:
    'App\Serializer\BookAttributeNormalizer':
        arguments: [ '@security.token_storage' ]
        tags:
            - { name: 'serializer.normalizer', priority: 64 }
```

The Normalizer class is a bit harder to understand, because it must ensure that it is only called once and that there is no recursion. 
To accomplish this, it needs to be aware of the parent Normalizer instance itself.

Here is an example:

```php
<?php
// api/src/Serializer/BookAttributeNormalizer.php

namespace App\Serializer;

use Symfony\Component\Security\Core\Authentication\Token\Storage\TokenStorageInterface;
use Symfony\Component\Serializer\Normalizer\ContextAwareNormalizerInterface;
use Symfony\Component\Serializer\Normalizer\NormalizerAwareInterface;
use Symfony\Component\Serializer\Normalizer\NormalizerAwareTrait;
use Symfony\Component\Serializer\Normalizer\NormalizerInterface;

class BookAttributeNormalizer implements ContextAwareNormalizerInterface, NormalizerAwareInterface
{
    use NormalizerAwareTrait;

    private const ALREADY_CALLED = 'BOOK_ATTRIBUTE_NORMALIZER_ALREADY_CALLED';

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

        $context[self::ALREADY_CALLED] = true;

        return $this->normalizer->normalize($object, $format, $context);
    }
    
    public function supportsNormalization($data, $format = null, array $context = [])
    {
        // Make sure we're not called twice
        if (isset($context[self::ALREADY_CALLED])) {
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
}
```

This will add the serialization group `can_retrieve_book` only if the currently logged-in user has access to the given book 
instance.

Note: In this example, we use the `TokenStorageInterface` to verify access to the book instance. However, Symfony 
provides many useful other services that might be better suited to your use case. For example, the [`AuthorizationChecker`](https://symfony.com/doc/current/components/security/authorization.html#authorization-checker).

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

## Decorating a Serializer and Adding Extra Data

In the following example, we will see how we add extra informations to the serialized output. Here is how we add the 
date on each request in `GET`:

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

API Platform is able to guess the entity identifier using Doctrine metadata ([ORM](http://doctrine-orm.readthedocs.org/en/latest/reference/basic-mapping.html#identifiers-primary-keys), [MongoDB ODM](https://www.doctrine-project.org/projects/doctrine-mongodb-odm/en/latest/reference/basic-mapping.html#identifiers)).
For ORM, it also supports composite identifiers.

If you are not using the Doctrine ORM or MongoDB ODM Provider, you must explicitly mark the identifier using the `identifier` attribute of
the `ApiPlatform\Core\Annotation\ApiProperty` annotation. For example:


```php
/**
 * @ApiResource()
 */
class Book
{
    // ...
    
    /**
     * @ApiProperty(identifier=true)
     */
    private $id;

    /**
     * This field can be managed only by an admin
     *
     * @var bool
     */
    public $active = false;

    /**
     * This field can be managed by any user
     *
     * @var string
     */
    public $name;

    // ...
}
```

You can also use the YAML configuration format:

```yaml
# api/config/api_platform/resources.yaml
App\Entity\Book:
    properties:
        id:
            identifier: true
```

In some cases, you will want to set the identifier of a resource from the client (e.g. a client-side generated UUID, or a slug).
In such cases, you must make the identifier property a writable class property. Specifically, to use client-generated IDs, you 
must do the following:

1. create a setter for the identifier of the entity (e.g. `public function setId(string $id)`) or make it a `public` property ,
2. add the denormalization group to the property (only if you use a specific denormalization group), and,
3. if you use Doctrine ORM, be sure to **not** mark this property with [the `@GeneratedValue` annotation](http://docs.doctrine-project.org/projects/doctrine-orm/en/latest/reference/basic-mapping.html#identifier-generation-strategies)
  or use the `NONE` value

## Embedding the JSON-LD Context

By default, the generated [JSON-LD context](https://www.w3.org/TR/json-ld/#the-context) (`@context`) is only referenced by
an IRI. A client that uses JSON-LD must send a second HTTP request to retrieve it:

```json
{
  "@context": "/contexts/Book",
  "@id": "/books/62",
  "@type": "Book",
  "name": "My awesome book",
  "author": "/people/59"
}
```

You can configure API Platform to embed the JSON-LD context in the root document by adding the `jsonld_embed_context`
attribute to the `@ApiResource` annotation:

```php
<?php
// api/src/Entity/Book.php

namespace App\Entity;

use ApiPlatform\Core\Annotation\ApiResource;

/**
 * @ApiResource(normalizationContext={"jsonld_embed_context"=true})
 */
class Book
{
    // ...
}
```

The JSON output will now include the embedded context:

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

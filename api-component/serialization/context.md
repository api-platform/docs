# The Serialization Context

Just like other Symfony and API Platform components, the Serializer component can be configured using annotations, XML
or YAML. Since annotations are easy to understand, we will use them in the following examples.

Note: if you aren't using the API Platform distribution, you will need to enable annotation support in the serializer configuration:

```yaml
# api/config/packages/framework.yaml
framework:
    serializer: { enable_annotations: true }
```

If you use [Symfony Flex](https://github.com/symfony/flex), just execute `composer req doctrine/annotations` and you are
all set!

If you want to use YAML or XML, please add the mapping path in the serializer configuration:

```yaml
# api/config/packages/framework.yaml
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
process. `denormalization_context` is passed as the 4th argument of [the `Serializer::deserialize()` method](https://api.symfony.com/master/Symfony/Component/Serializer/SerializerInterface.html#method_deserialize) during denormalization (writing).

To configure the serialization groups of classes's properties, you must use directly [the Symfony Serializer's configuration files or annotations](https://symfony.com/doc/current/components/serializer.html#attributes-groups).


In addition to the `groups` key, you can configure any Symfony Serializer option through the `$context` parameter
(e.g. the `enable_max_depth`key when using [the `@MaxDepth` annotation](https://symfony.com/doc/current/components/serializer.html#handling-serialization-depth)).

Any serialization and deserialization group that you specify will also be leveraged by the built-in actions and the Hydra
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

Refer to the [operations](../usage-and-configuration/operations.md) documentation to learn more.

### Advanced usage

As you can see, setting the appropriate the normalization / denormalization context just requires some configuration. 
But in some cases, as your application grows, you're likely to adjust those settings depending on many factors:

* The resource currently being serialized
* The owner of this resource
* The user's permissions
* ...

To achieve this, learn [how to change the Serialization Context dynamically](dynamic-context.md).

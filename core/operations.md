# Operations

API Platform Core relies on the concept of operations. Operations can be applied to a resource exposed by the API. From
an implementation point of view, an operation is a link between a resource, a route and its related controller.

API Platform automatically registers typical [CRUD](https://en.wikipedia.org/wiki/Create,_read,_update_and_delete) operations
and describes them in the exposed documentation (Hydra and Swagger). It also creates and registers routes corresponding
to these operations in the Symfony routing system (if it is available).

The behavior of built-in operations is briefly presented in the [Getting started](getting-started.md#mapping-the-entities)
guide.

The list of enabled operations can be configured on a per-resource basis. Creating custom operations on specific routes
is also possible.

There are two types of operations: collection operations and item operations.

Collection operations act on a collection of resources. By default two routes are implemented: `POST` and `GET`. Item
operations act on an individual resource. Three default routes are defined: `GET`, `PUT` and `DELETE` (`PATCH` is also supported
when [using the JSON API format](content-negotiation.md), as required by the specification).

When the `ApiPlatform\Core\Annotation\ApiResource` annotation is applied to an entity class, the following built-in CRUD
operations are automatically enabled:

*Collection operations*

Method | Mandatory | Description
-------|-----------|------------------------------------------
`GET`  | yes       | Retrieve the (paginated) list of elements
`POST` | no        | Create a new element

*Item operations*

Method   | Mandatory | Description
---------|-----------|-------------------------------------------
`GET`    | yes       | Retrieve an element
`PUT`    | no        | Replace an element
`PATCH`  | no        | Apply a partial modification to an element
`DELETE` | no        | Delete an element

Note: the `PATCH` method must be enabled explicitly in the configuration, refer to the [Content Negotiation](content-negotiation.md) section for more information.

## Enabling and Disabling Operations

If no operation is specified, all default CRUD operations are automatically registered. It is also possible - and recommended
for large projects - to define operations explicitly.

Keep in mind that `collectionOperations` and `itemOperations` behave independently. For instance, if you don't explicitly
configure operations for `collectionOperations`, `GET` and `POST` operations will be automatically registered, even if you
explicitly configure `itemOperations`. The reverse is also true.

Operations can be configured using annotations, XML or YAML. In the following examples, we enable only the built-in operation
for the `GET` method for both `collectionOperations` and `itemOperations` to create a readonly endpoint.

`itemOperations` and `collectionOperations` are arrays containing a list of operation. Each operation is defined by a key
corresponding to the name of the operation that can be anything you want and an array of properties as value. If an
empty list of operations is provided, all operations are disabled.

If the operation's name match a supported HTTP methods (`GET`, `POST`, `PUT`, `PATCH` or `DELETE`), the corresponding `method` property
will be automatically added.

```php
<?php
// api/src/Entity/Book.php

namespace App\Entity;

use ApiPlatform\Core\Annotation\ApiResource;

/**
 * ...
 * @ApiResource(
 *     collectionOperations={"get"},
 *     itemOperations={"get"}
 * )
 */
class Book
{
    // ...
}
```

The previous example can also be written with an explicit method definition:

```php
<?php
// api/src/Entity/Book.php

namespace App\Entity;

use ApiPlatform\Core\Annotation\ApiResource;

/**
 * ...
 * @ApiResource(
 *     collectionOperations={"get"={"method"="GET"}},
 *     itemOperations={"get"={"method"="GET"}}
 * )
 */
class Book
{
    // ...
}
```

Alternatively, you can use the YAML configuration format:

```yaml
# api/config/api_platform/resources.yaml
App\Entity\Book:
    collectionOperations:
        get: ~ # nothing more to add if we want to keep the default controller
    itemOperations:
        get: ~
```

Or the XML configuration format:

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<!-- api/config/api_platform/resources.xml -->

<resources xmlns="https://api-platform.com/schema/metadata"
           xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
           xsi:schemaLocation="https://api-platform.com/schema/metadata
           https://api-platform.com/schema/metadata/metadata-2.0.xsd">
    <resource class="App\Entity\Book">
        <itemOperations>
            <itemOperation name="get" />
        </itemOperations>
        <collectionOperations>
            <collectionOperation name="get" />
        </collectionOperations>
    </resource>
</resources>
```

API Platform Core is smart enough to automatically register the applicable Symfony route referencing a built-in CRUD action
just by specifying the method name as key, or by checking the explicitly configured HTTP method.

If you do not want to allow access to the resource item (i.e. you don't want a `GET` item operation), instead of omitting it altogether, you should instead declare a `GET` item operation which returns HTTP 404 (Not Found), so that the resource item can still be identified by an IRI. For example:

```
<?php
// api/src/Entity/Book.php

namespace App\Entity;

use ApiPlatform\Core\Action\NotFoundAction;
use ApiPlatform\Core\Annotation\ApiResource;

/**
 * @ApiResource(
 *     collectionOperations={
 *         "get",
 *     },
 *     itemOperations={
 *         "get"={
 *             "controller"=NotFoundAction::class,
 *             "read"=false,
 *             "output"=false,
 *         },
 *     },
 * )
 */
 class Book
 {
 }
```

## Configuring Operations

The URL, the method and the default status code (among other options) can be configured per operation.

In the next example, both `GET` and `POST` operations are registered with custom URLs. Those will override the URLs generated by default.
In addition to that, we require the `id` parameter in the URL of the `GET` operation to be an integer, and we configure the status code generated after successful `POST` request to be `301`

```php
<?php
// api/src/Entity/Book.php

namespace App\Entity;

use ApiPlatform\Core\Annotation\ApiResource;

/**
 * ...
 * @ApiResource(
 *     collectionOperations={
 *         "post"={"path"="/grimoire", status=301}
 *     },
 *     itemOperations={
 *         "get"={
 *             "path"="/grimoire/{id}",
 *             "requirements"={"id"="\d+"},
 *             "defaults"={"color"="brown"},
 *             "options"={"my_option"="my_option_value"},
 *             "schemes"={"https"},
 *             "host"="{subdomain}.api-platform.com"
 *         }
 *     }
 * )
 */
class Book
{
    //...
}
```

Or in YAML:

```yaml
# api/config/api_platform/resources.yaml
App\Entity\Book:
    collectionOperations:
        post:
            path: '/grimoire'
            status: 301
    itemOperations:
        get:
            method: 'GET'
            path: '/grimoire/{id}'
            requirements:
                id: '\d+'
            defaults:
                color: 'brown'
            host: '{subdomain}.api-platform.com'
            schemes: ['https']
            options:
                my_option: 'my_option_value'
```

Or in XML:

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<!-- api/config/api_platform/resources.xml -->

<resources xmlns="https://api-platform.com/schema/metadata"
           xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
           xsi:schemaLocation="https://api-platform.com/schema/metadata
           https://api-platform.com/schema/metadata/metadata-2.0.xsd">
    <resource class="App\Entity\Book">
        <collectionOperations>
            <collectionOperation name="post">
                <attribute name="path">/grimoire</attribute>
                <attribute name="status">301</attribute>
            </collectionOperation>
        </collectionOperations>
        <itemOperations>
            <itemOperation name="get">
                <attribute name="path">/grimoire/{id}</attribute>
                <attribute name="requirements">
                    <attribute name="id">\d+</attribute>
                </attribute>
                <attribute name="defaults">
                    <attribute name="color">brown</attribute>
                </attribute>
                <attribute name="host">{subdomain}.api-platform.com</attribute>
                <attribute name="schemes">
                    <attribute>https</attribute>
                </attribute>
                <attribute name="options">
                    <attribute name="color">brown</attribute>
                </attribute>
            </itemOperation>
        </itemOperations>
    </resource>
</resources>
```

In all these examples, the `method` attribute is omitted because it matches the operation name.

## Prefixing All Routes of All Operations

Sometimes it's also useful to put a whole resource into its own "namespace" regarding the URI. Let's say you want to
put everything that's related to a `Book` into the `library` so that URIs become `library/book/{id}`. In that case
you don't need to override all the operations to set the path but configure the `route_prefix` attribute for the whole entity instead:

```php
<?php
// api/src/Entity/Book.php

namespace App\Entity;

use ApiPlatform\Core\Annotation\ApiResource;

/**
 * @ApiResource(routePrefix="/library")
 */
class Book
{
    //...
}
```

Alternatively, the more verbose attribute syntax can be used `@ApiResource(attributes={"route_prefix"="/library"})`

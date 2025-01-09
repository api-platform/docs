# Operations

API Platform Core relies on the concept of operations. Operations can be applied to a resource exposed by the API. From
an implementation point of view, an operation is a link between a resource, a route and its related controller.

<p align="center" class="symfonycasts"><a href="https://symfonycasts.com/screencast/api-platform/operations?cid=apip"><img src="../distribution/images/symfonycasts-player.png" alt="Operations screencast"><br>Watch the Operations screencast</a></p>

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
when [using the JSON:API format](content-negotiation.md), as required by the specification).

When the `ApiPlatform\Core\Annotation\ApiResource` annotation is applied to an entity class, the following built-in CRUD
operations are automatically enabled:

Collection operations:

Method | Mandatory | Description
-------|-----------|------------------------------------------
`GET`  | yes       | Retrieve the (paginated) list of elements
`POST` | no        | Create a new element

Item operations:

Method   | Mandatory | Description
---------|-----------|-------------------------------------------
`GET`    | yes       | Retrieve an element
`PUT`    | no        | Replace an element
`PATCH`  | no        | Apply a partial modification to an element
`DELETE` | no        | Delete an element

Note: the `PATCH` method must be enabled explicitly in the configuration, refer to the [Content Negotiation](content-negotiation.md) section for more information.

Note: with JSON Merge Patch, the [null values will be skipped](https://symfony.com/doc/current/components/serializer.html#skipping-null-values) in the response.

Note: Current `PUT` implementation behaves more or less like the `PATCH` method.
Existing properties not included in the payload are **not** removed, their current values are preserved.
To remove an existing property, its value must be explicitly set to `null`.
Implementing [the standard `PUT` behavior](https://httpwg.org/specs/rfc7231.html#PUT) is on the roadmap, follow [issue #4344](https://github.com/api-platform/core/issues/4344) to track the progress.

## Enabling and Disabling Operations

If no operation is specified, all default CRUD operations are automatically registered. It is also possible - and recommended
for large projects - to define operations explicitly.

Keep in mind that `collectionOperations` and `itemOperations` behave independently. For instance, if you don't explicitly
configure operations for `collectionOperations`, `GET` and `POST` operations will be automatically registered, even if you
explicitly configure `itemOperations`. The reverse is also true.

Operations can be configured using annotations, XML or YAML. In the following examples, we enable only the built-in operation
for the `GET` method for both `collectionOperations` and `itemOperations` to create a readonly endpoint.

`itemOperations` and `collectionOperations` are arrays containing a list of operations. Each operation is defined by a key
corresponding to the name of the operation that can be anything you want and an array of properties as value. If an
empty list of operations is provided, all operations are disabled.

If the operation's name matches a supported HTTP methods (`GET`, `POST`, `PUT`, `PATCH` or `DELETE`), the corresponding `method` property
will be automatically added.

<code-selector>

```php
<?php
// api/src/Entity/Book.php

namespace App\Entity;

use ApiPlatform\Core\Annotation\ApiResource;

#[ApiResource(
    collectionOperations: ['get'],
    itemOperations: ['get'],
)]
class Book
{
    // ...
}
```

```yaml
# config/api_platform/resources.yaml
App\Entity\Book:
    collectionOperations:
        get: ~ # nothing more to add if we want to keep the default controller
    itemOperations:
        get: ~
```

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

</code-selector>

The previous example can also be written with an explicit method definition:

<code-selector>

```php
<?php
// api/src/Entity/Book.php

namespace App\Entity;

use ApiPlatform\Core\Annotation\ApiResource;

#[ApiResource(
    collectionOperations: [
        'get' => ['method' => 'get'],
    ],
    itemOperations: [
        'get' => ['method' => 'get'],
    ],
)]
class Book
{
    // ...
}
```

```yaml
# config/api_platform/resources.yaml
App\Entity\Book:
    collectionOperations:
        get:
            method: GET
    itemOperations:
        get:
            method: GET
```

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<!-- api/config/api_platform/resources.xml -->

<resources xmlns="https://api-platform.com/schema/metadata"
        xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:schemaLocation="https://api-platform.com/schema/metadata
        https://api-platform.com/schema/metadata/metadata-2.0.xsd">
    <resource class="App\Entity\Book">
        <collectionOperations>
            <collectionOperation name="get" />
        </collectionOperations>
        <itemOperations>
            <itemOperation name="get">
                <attribute name="method">GET</attribute>
            </itemOperation>
        </itemOperations>
    </resource>
</resources>
```

</code-selector>

API Platform Core is smart enough to automatically register the applicable Symfony route referencing a built-in CRUD action
just by specifying the method name as key, or by checking the explicitly configured HTTP method.

If you do not want to allow access to the resource item (i.e. you don't want a `GET` item operation), instead of omitting it altogether, you should instead declare a `GET` item operation which returns HTTP 404 (Not Found), so that the resource item can still be identified by an IRI. For example:

<code-selector>

```php
<?php
// api/src/Entity/Book.php

namespace App\Entity;

use ApiPlatform\Core\Action\NotFoundAction;
use ApiPlatform\Core\Annotation\ApiResource;

#[ApiResource(
    collectionOperations: [
        'get' => ['method' => 'get'],
    ],
    itemOperations: [
        'get' => [
            'controller' => NotFoundAction::class,
            'read' => false,
            'output' => false,
        ],
    ],
)]
class Book
{
}
```

```yaml
# config/api_platform/resources.yaml
App\Entity\Book:
    collectionOperations:
        get: ~
    itemOperations:
        get:
            controller: ApiPlatform\Core\Action\NotFoundAction
            read: false
            output: false
```

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<!-- api/config/api_platform/resources.xml -->

<resources xmlns="https://api-platform.com/schema/metadata"
        xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:schemaLocation="https://api-platform.com/schema/metadata
        https://api-platform.com/schema/metadata/metadata-2.0.xsd">
    <resource class="App\Entity\Book">
        <collectionOperations>
            <collectionOperation name="get" />
        </collectionOperations>
        <itemOperations>
            <itemOperation name="get">
                <attribute name="controller">ApiPlatform\Core\Action\NotFoundAction</attribute>
                <attribute name="read">false</attribute>
                <attribute name="output">false</attribute>
            </itemOperation>
        </itemOperations>
    </resource>
</resources>
```

</code-selector>

## Configuring Operations

The URL, the method and the default status code (among other options) can be configured per operation.

In the next example, both `GET` and `POST` operations are registered with custom URLs. Those will override the URLs generated by default.
In addition to that, we require the `id` parameter in the URL of the `GET` operation to be an integer, and we configure the status code generated after successful `POST` request to be `301`:

<code-selector>

```php
<?php
// api/src/Entity/Book.php

namespace App\Entity;

use ApiPlatform\Core\Annotation\ApiResource;

#[ApiResource(
    collectionOperations: [
        'post' => [
            'path' => '/grimoire',
            'status' => 301,
        ],
    ],
    itemOperations: [
        'get' => [
            'path' => '/grimoire/{id}',
            'requirements' => ['id' => '\d+'],
            'defaults' => ['color' => 'brown'],
            'options' => ['my_option' => 'my_option_value'],
            'schemes' => ['https'],
            'host' => '{subdomain}.api-platform.com',
        ],
    ],
)]
class Book
{
    //...
}
```

```yaml
# config/api_platform/resources.yaml
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

</code-selector>

In all these examples, the `method` attribute is omitted because it matches the operation name.

When specifying sub options, you must always use snake case as demonstrated below with the `denormalization_context` option on the `put` operation:

<code-selector>

```php
<?php
// api/src/Entity/Book.php

namespace App\Entity;

use ApiPlatform\Core\Annotation\ApiResource;

#[ApiResource(
    itemOperations: [
            'get',
            'put' => [
                'denormalization_context' => [
                    'groups' => ['item:put'],
                    'swagger_definition_name' => 'put',
                ],
            ],
            'delete',
        ],
    ],
    denormalizationContext: [
        'groups' => ['item:post'],
        'swagger_definition_name' => 'post',
    ],
)]
class Book
{
    //...
}
```

```yaml
# config/api_platform/resources.yaml
App\Entity\Book:
    itemOperations:
        get: ~
        put:
            denormalization_context:
                groups: ['item:put']
                swagger_definition_name: 'put',
        delete: ~
    denormalizationContext:
        groups: ['item:post']
        swagger_definition_name: 'post'
```

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
            <itemOperation name="put">
                <attribute name="denormalization_context">
                    <attribute name="groups">
                        <attribute>item:put</attribute>
                    </attribute>
                    <attribute name="swagger_definition_name">put</attribute>
                </attribute>
            </itemOperation>
            <itemOperation name="delete" />
        </itemOperations>
        <denormalizationContext>
            <attribute name="groups">
                <attribute>item:post</attribute>
            </attribute>
            <attribute name="swagger_definition_name">post</attribute>
        </denormalizationContext>
    </resource>
</resources>
```

</code-selector>

## Prefixing All Routes of All Operations

Sometimes it's also useful to put a whole resource into its own "namespace" regarding the URI. Let's say you want to
put everything that's related to a `Book` into the `library` so that URIs become `library/book/{id}`. In that case
you don't need to override all the operations to set the path but configure the `route_prefix` attribute for the whole entity instead:

<code-selector>

```php
<?php
// api/src/Entity/Book.php

namespace App\Entity;

use ApiPlatform\Core\Annotation\ApiResource;

#[ApiResource(routePrefix: '/library')]
class Book
{
    //...
}
```

```yaml
# config/api_platform/resources.yaml
App\Entity\Book:
    attributes:
        route_prefix: /library
```

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<!-- api/config/api_platform/resources.xml -->

<resources xmlns="https://api-platform.com/schema/metadata"
        xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:schemaLocation="https://api-platform.com/schema/metadata
        https://api-platform.com/schema/metadata/metadata-2.0.xsd">
    <resource class="App\Entity\Book">
        <attribute name="route_prefix">/library</attribute>
    </resource>
</resources>
```

</code-selector>

Alternatively, the more verbose attribute syntax can be used: `#[ApiResource(attributes: ["route_prefix" => "/library"])]`.

## Expose a Model Without Any Routes

Sometimes, you may want to expose a model, but want it to be used through subrequests only, and never through item or collection operations.
Because the OpenAPI standard requires at least one route to be exposed to make your models consumable, let's see how you can manage this kind
of issue.

Let's say you have the following entities in your project:

```php
<?php
// src/Entity/Place.php

namespace App\Entity;

use Doctrine\ORM\Mapping as ORM;

#[ORM\Entity]
class Place
{
    #[ORM\Id, ORM\Column, ORM\GeneratedValue]
    private ?int $id = null;

    #[ORM\Column] 
    private string $name = '';

    #[ORM\Column(type: 'float')]
    private float $latitude = 0;

    #[ORM\Column(type: 'float')] 
    private float $longitude = 0;

    // ...
}
```

```php
<?php
// src/Entity/Weather.php

namespace App\Entity;

class Weather
{
    private float $temperature;

    private float $pressure;

    // ...
}
```

We don't save the `Weather` entity in the database, since we want to return the weather in real time when it is queried.
Because we want to get the weather for a known place, it is more reasonable to query it through a subresource of the `Place` entity, so let's do this:

```php
<?php
// src/Entity/Place.php

namespace App\Entity;

use ApiPlatform\Core\Annotation\ApiResource;
use App\Controller\GetWeather;
use Doctrine\ORM\Mapping as ORM;

#[ORM\Entity]
#[ApiResource(
    collectionOperations: [
        'get',
        'post',
    ],
    itemOperations: [
        'get',
        'put',
        'delete',
        'get_weather' => [
            'method' => 'GET',
            'path' => '/places/{id}/weather',
            'controller' => GetWeather::class,
        ],
    ],
)]
class Place
{
    // ...
```

The `GetWeather` controller fetches the weather for the given city and returns an instance of the `Weather` entity.
This implies that API Platform has to know about this entity, so we will need to make it an API resource too:

```php
<?php
// src/Entity/Weather.php

namespace App\Entity;

use ApiPlatform\Core\Annotation\ApiResource;

#[ApiResource]
class Weather
{
    // ...
```

This will expose the `Weather` model, but also all the default CRUD routes: `GET`, `PUT`, `PATCH`, `DELETE` and `POST`, which is a non-sense in our context.
Since we are required to expose at least one route, let's expose just one:

```php
<?php
// src/Entity/Weather.php

namespace App\Entity;

use ApiPlatform\Core\Annotation\ApiResource;

#[ApiResource(
    itemOperations: [
        'get' => [
            'method' => 'GET',
            'controller' => SomeRandomController::class,
        ],
    ],
)]
class Weather
{
    // ...
```

This way, we expose a route that will do… nothing. Note that the controller does not even need to exist.

It's almost done, we have just one final issue: our fake item operation is visible in the API docs.
To remove it, we will need to [decorate the Swagger documentation](openapi.md#overriding-the-openapi-specification).
Then, remove the route from the decorator:

```php
<?php
// src/OpenApi/OpenApiFactory.php

namespace App\OpenApi;

use ApiPlatform\Core\OpenApi\Factory\OpenApiFactoryInterface;
use ApiPlatform\Core\OpenApi\OpenApi;
use ApiPlatform\Core\OpenApi\Model;

final class OpenApiFactory implements OpenApiFactoryInterface
{
    private $decorated;

    public function __construct(OpenApiFactoryInterface $decorated)
    {
        $this->decorated = $decorated;
    }

    public function __invoke(array $context = []): OpenApi
    {
        $openApi = $this->decorated->__invoke($context);

        $paths = $openApi->getPaths()->getPaths();

        $filteredPaths = new Model\Paths();
        foreach ($paths as $path => $pathItem) {
            // If a prefix is configured on API Platform's routes, it must appear here.
            if ($path === '/weathers/{id}') {
                continue;
            }
            $filteredPaths->addPath($path, $pathItem);
        }

        return $openApi->withPaths($filteredPaths);
    }
}
```

That's it: your route is gone!

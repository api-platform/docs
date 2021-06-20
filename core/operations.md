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

[codeSelector]

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
# api/config/api_platform/resources.yaml
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

[/codeSelector]

The previous example can also be written with an explicit method definition:

[codeSelector]

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
# api/config/api_platform/resources.yaml
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

[/codeSelector]

API Platform Core is smart enough to automatically register the applicable Symfony route referencing a built-in CRUD action
just by specifying the method name as key, or by checking the explicitly configured HTTP method.

If you do not want to allow access to the resource item (i.e. you don't want a `GET` item operation), instead of omitting it altogether, you should instead declare a `GET` item operation which returns HTTP 404 (Not Found), so that the resource item can still be identified by an IRI. For example:

[codeSelector]

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
# api/config/api_platform/resources.yaml
App\Entity\Book:
    collectionOperations:
        get: ~
    itemOperations:
        get:
            controller: App\Controller\NotFoundAction
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
                <attribute name="controller">App\Controller\NotFoundAction</attribute>
                <attribute name="read">false</attribute>
                <attribute name="output">false</attribute>
            </itemOperation>
        </itemOperations>
    </resource>
</resources>
```

[/codeSelector]

## Configuring Operations

The URL, the method and the default status code (among other options) can be configured per operation.

In the next example, both `GET` and `POST` operations are registered with custom URLs. Those will override the URLs generated by default.
In addition to that, we require the `id` parameter in the URL of the `GET` operation to be an integer, and we configure the status code generated after successful `POST` request to be `301`:

[codeSelector]

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

[/codeSelector]

In all these examples, the `method` attribute is omitted because it matches the operation name.

## Prefixing All Routes of All Operations

Sometimes it's also useful to put a whole resource into its own "namespace" regarding the URI. Let's say you want to
put everything that's related to a `Book` into the `library` so that URIs become `library/book/{id}`. In that case
you don't need to override all the operations to set the path but configure the `route_prefix` attribute for the whole entity instead:

[codeSelector]

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
# api/config/api_platform/resources.yaml
App\Entity\Book:
    attributes:
        route_prefix: /library
    itemOperations:
        get: ~
        post_publication:
            route_name: book_post_publication
        book_post_discontinuation: ~
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
            <itemOperation name="post_publication">
                <attribute name="route_name">book_post_publication</attribute>
            </itemOperation>
            <itemOperation name="book_post_discontinuation" />
        </itemOperations>
    </resource>
</resources>
```

[/codeSelector]

Alternatively, the more verbose attribute syntax can be used: `#[ApiResource(attributes: ["route_prefix"=>"/library"])]`.

API Platform will automatically map this `post_publication` operation to the route `book_post_publication`. Let's create a custom action
and its related route using annotations:

```php
<?php
// api/src/Controller/CreateBookPublication.php

namespace App\Controller;

use App\Entity\Book;
use Symfony\Component\Routing\Annotation\Route;

class CreateBookPublication
{
    public function __construct(
        private BookPublishingHandler $bookPublishingHandler
    ) {}
    
    #[Route(
        path: '/books/{id}/publication',
        name: 'book_post_publication',
        defaults: [
            '_api_resource_class' => Book::class,
            '_api_item_operation_name' => 'post_publication',
        ],
        methods: ['POST'],
    )]
    public function __invoke(Book $data): Book
    {
        $this->bookPublishingHandler->handle($data);

        return $data;
    }
}
```

It is mandatory to set `_api_resource_class` and `_api_item_operation_name` (or `_api_collection_operation_name` for a collection
operation) in the parameters of the route (`defaults` key). It allows API Platform to work with the Symfony routing system.

Alternatively, you can also use a traditional Symfony controller and YAML or XML route declarations. The following example does
the exact same thing as the previous example:

```php
<?php
// api/src/Controller/BookController.php

namespace App\Controller;

use App\Entity\Book;
use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;

class BookController extends AbstractController
{
    public function createPublication(Book $data, BookPublishingHandler $bookPublishingHandler): Book
    {
        return $bookPublishingHandler->handle($data);
    }
}
```

```yaml
# api/config/routes.yaml
book_post_publication:
    path: /books/{id}/publication
    methods: ['POST']
    defaults:
        _controller: App\Controller\BookController::createPublication
        _api_resource_class: App\Entity\Book
        _api_item_operation_name: post_publication
```

## Expose a model without any routes

Sometimes, you may want to expose a model, but want it to be used through subrequests only, and never through item or collection operations.
Because the OpenAPI standard requires at least one route to be exposed to make your models consumable, let's see how you can manage this kind
of issue.

Let's say you have the following entities in your project:

```php
<?php
// src/Entity/Place.php

namespace App\Entity;

use Doctrine\ORM\Mapping as ORM;

/**
 * @ORM\Entity
 */
class Place
{
    /**
     * @ORM\Id
     * @ORM\GeneratedValue
     * @ORM\Column(type="integer")
     */
    private ?int $id = null;

    /**
     * @ORM\Column
     */
    private string $name = '';

    /**
     * @ORM\Column(type="float")
     */
    private float $latitude = 0;

    /**
     * @ORM\Column(type="float")
     */
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

/**
 * @ORM\Entity
 */
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

This will expose the `Weather` model, but also all the default CRUD routes: `GET`, `PUT`, `DELETE` and `POST`, which is a non-sense in our context.
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
To remove it, we will need to [decorate the Swagger documentation](swagger.md#overriding-the-openapi-specification).
Then, remove the route from the decorator:

```php
<?php
// src/Swagger/SwaggerDecorator.php

namespace App\Swagger;

use Symfony\Component\Serializer\Normalizer\NormalizerInterface;

final class SwaggerDecorator implements NormalizerInterface
{
    public function __construct(
        private NormalizerInterface $decorated
    ) {}

    public function normalize($object, string $format = null, array $context = [])
    {
        $docs = $this->decorated->normalize($object, $format, $context);

        // If a prefix is configured on API Platform's routes, it must appear here.
        unset($docs['paths']['/weathers/{id}']);

        return $docs;
    }

    // ...
```

That's it: your route is gone!

# Operations

API Platform relies on the concept of operations. Operations can be applied to a resource exposed by the API. From
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
operations act on an individual resource. Four default routes are defined: `GET`, `PUT`, `DELETE` and `PATCH`. `PATCH` is supported
with [JSON Merge Patch (RFC 7396)](https://www.rfc-editor.org/rfc/rfc7386), or [using the JSON:API format](https://jsonapi.org/format/#crud-updating), as required by the specification.

When the `ApiPlatform\Metadata\ApiResource` annotation is applied to an entity class, the following built-in CRUD
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

Keep in mind that once you explicitly set up an operation, the automatically registered CRUD will no longer be.
If you declare even one operation manually, such as `#[GET]`, you must declare the others manually as well if you need them.

Operations can be configured using annotations, XML or YAML. In the following examples, we enable only the built-in operation
for the `GET` method for both `collection` and `item` to create a readonly endpoint.

If the operation's name matches a supported HTTP methods (`GET`, `POST`, `PUT`, `PATCH` or `DELETE`), the corresponding `method` property
will be automatically added.

Note: The `#[GetCollection]` attribute is an alias for `#[Get(collection: true)]`

<code-selector>

```php
<?php
// api/src/Entity/Book.php
namespace App\Entity;

use ApiPlatform\Metadata\ApiResource;
use ApiPlatform\Metadata\Get;
use ApiPlatform\Metadata\GetCollection;

#[ApiResource(
    operations: [
        new Get(),
        new GetCollection()
    ]
)]
class Book
{
    // ...
}
```

```yaml
# api/config/api_platform/resources.yaml
App\Entity\Book:
    operations:
        ApiPlatform\Metadata\GetCollection: ~ # nothing more to add if we want to keep the default controller
        ApiPlatform\Metadata\Get: ~
```

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<!-- api/config/api_platform/resources.xml -->

<resources xmlns="https://api-platform.com/schema/metadata/resources-3.0"
        xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:schemaLocation="https://api-platform.com/schema/metadata/resources-3.0
        https://api-platform.com/schema/metadata/resources-3.0.xsd">
    <resource class="App\Entity\Book">
        <operations>
            <operation class="ApiPlatform\Metadata\Get" />
            <operation class="ApiPlatform\Metadata\GetCollection" />
        </operations>
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

use ApiPlatform\Metadata\ApiResource;
use ApiPlatform\Metadata\Get;
use ApiPlatform\Metadata\GetCollection;

#[ApiResource(
    operations: [
        new Get(),
        new GetCollection()
    ]
)]
class Book
{
    // ...
}
```

```yaml
# api/config/api_platform/resources.yaml
App\Entity\Book:
    operations:
        ApiPlatform\Metadata\GetCollection:
            method: GET
        ApiPlatform\Metadata\Get:
            method: GET
```

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<!-- api/config/api_platform/resources.xml -->

<resources xmlns="https://api-platform.com/schema/metadata/resources-3.0"
        xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:schemaLocation="https://api-platform.com/schema/metadata/resources-3.0
        https://api-platform.com/schema/metadata/resources-3.0.xsd">
    <resource class="App\Entity\Book">
        <operations>
            <operation class="ApiPlatform\Metadata\GetCollection" />
            <operation class="ApiPlatform\Metadata\Get" method="GET" />
        </operations>
    </resource>
</resources>
```

</code-selector>

API Platform is smart enough to automatically register the applicable Symfony route referencing a built-in CRUD action
just by specifying the method name as key, or by checking the explicitly configured HTTP method.

If you do not want to allow access to the resource item (i.e. you don't want a `GET` item operation), instead of omitting it altogether, you should instead declare a `GET` item operation which returns HTTP 404 (Not Found), so that the resource item can still be identified by an IRI. For example:

<code-selector>

```php
<?php
// api/src/Entity/Book.php
namespace App\Entity;

use ApiPlatform\Action\NotFoundAction;
use ApiPlatform\Metadata\Get;
use ApiPlatform\Metadata\GetCollection;
use ApiPlatform\Metadata\ApiResource;

#[ApiResource(operations: [
    new Get(
        controller: NotFoundAction::class, 
        read: false, 
        output: false
    ),
    new GetCollection()
])]
class Book
{
    // ...
}
```

```yaml
# api/config/api_platform/resources.yaml
App\Entity\Book:
    operations:
        ApiPlatform\Metadata\GetCollection: ~
        ApiPlatform\Metadata\Get:
            controller: ApiPlatform\Action\NotFoundAction
            read: false
            output: false
```

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<!-- api/config/api_platform/resources.xml -->

<resources xmlns="https://api-platform.com/schema/metadata/resources-3.0"
        xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:schemaLocation="https://api-platform.com/schema/metadata/resources-3.0
        https://api-platform.com/schema/metadata/resources-3.0.xsd">
    <resource class="App\Entity\Book">
        <operations>
            <operation class="ApiPlatform\Metadata\GetCollection" />
            <operation class="ApiPlatform\Metadata\Get" controller="ApiPlatform\Action\NotFoundAction"
                       read="false" output="false" />
        </operations>
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

use ApiPlatform\Metadata\ApiResource;
use ApiPlatform\Metadata\Get;
use ApiPlatform\Metadata\Post;

#[ApiResource(operations: [
    new Get(
        uriTemplate: '/grimoire/{id}', 
        requirements: ['id' => '\d+'], 
        defaults: ['color' => 'brown'], 
        options: ['my_option' => 'my_option_value'], 
        schemes: ['https'], 
        host: '{subdomain}.api-platform.com'
    ),
    new Post(
        uriTemplate: '/grimoire', 
        status: 301
    )
])]
class Book
{
    //...
}
```

```yaml
# api/config/api_platform/resources.yaml
App\Entity\Book:
    operations:
        ApiPlatform\Metadata\Post:
            uriTemplate: '/grimoire'
            status: 301
        ApiPlatform\Metadata\Get:
            uriTemplate: '/grimoire/{id}'
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

<resources xmlns="https://api-platform.com/schema/metadata/resources-3.0"
        xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:schemaLocation="https://api-platform.com/schema/metadata/resources-3.0
        https://api-platform.com/schema/metadata/resources-3.0.xsd">
    <resource class="App\Entity\Book">
        <operations>
            <operation class="ApiPlatform\Metadata\Post" uriTemplate="/grimoire" status="301" />
            <operation class="ApiPlatform\Metadata\Get" uriTemplate="/grimoire/{id}" host="{subdomain}.api-platform.com">
                <requirements>
                    <requirement property="id">\d+</requirement>
                </requirements>
                <defaults>
                    <values>
                        <value name="color">brown</value>
                    </values>
                </defaults>
                <schemes>
                    <scheme>https</scheme>
                </schemes>
                <options>
                    <values>
                        <value name="color">brown</value>
                    </values>
                </options>
            </operation>
        </operations>
    </resource>
</resources>
```

</code-selector>

## Prefixing All Routes of All Operations

Sometimes it's also useful to put a whole resource into its own "namespace" regarding the URI. Let's say you want to
put everything that's related to a `Book` into the `library` so that URIs become `library/book/{id}`. In that case
you don't need to override all the operations to set the path but configure the `routePrefix` attribute for the whole entity instead:

<code-selector>

```php
<?php
// api/src/Entity/Book.php
namespace App\Entity;

use ApiPlatform\Metadata\ApiResource;

#[ApiResource(routePrefix: '/library')]
class Book
{
    //...
}
```

```yaml
# api/config/api_platform/resources.yaml
App\Entity\Book:
    routePrefix: /library
```

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<!-- api/config/api_platform/resources.xml -->

<resources xmlns="https://api-platform.com/schema/metadata/resources-3.0"
        xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:schemaLocation="https://api-platform.com/schema/metadata/resources-3.0
        https://api-platform.com/schema/metadata/resources-3.0.xsd">
    <resource class="App\Entity\Book" routePrefix="/library" />
</resources>
```

</code-selector>

API Platform will automatically map this `post_publication` operation to the route `book_post_publication`. Let's create a custom action
and its related route using annotations:

```php
<?php
// api/src/Controller/CreateBookPublication.php

namespace App\Controller;

use App\Entity\Book;
use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
use Symfony\Component\HttpKernel\Attribute\AsController;
use Symfony\Component\Routing\Annotation\Route;

#[AsController]
class CreateBookPublication extends AbstractController
{
    public function __construct(
        private BookPublishingHandler $bookPublishingHandler
    ) {}

    #[Route(
        path: '/books/{id}/publication',
        name: 'book_post_publication',
        defaults: [
            '_api_resource_class' => Book::class,
            '_api_operation_name' => '_api_/books/{id}/publication_post',
        ],
        methods: ['POST'],
    )]
    public function __invoke(Book $book): Book
    {
        $this->bookPublishingHandler->handle($book);

        return $book;
    }
}
```

It is mandatory to set `_api_resource_class` and `_api_operation_name`in the parameters of the route (`defaults` key). It allows API Platform to work with the Symfony routing system.

Alternatively, you can also use a traditional Symfony controller and YAML or XML route declarations. The following example does
the exact same thing as the previous example:

```php
<?php
// api/src/Controller/BookController.php

namespace App\Controller;

use App\Entity\Book;
use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
use Symfony\Component\HttpKernel\Attribute\AsController;

#[AsController]
class BookController extends AbstractController
{
    public function createPublication(Book $book, BookPublishingHandler $bookPublishingHandler): Book
    {
        return $bookPublishingHandler->handle($book);
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
        _api_operation_name: post_publication
```

## Defining Which Operation to Use to Generate the IRI

Using multiple operations on your resource, you may want to specify which operation to use to generate the IRI, instead
of letting API Platform use the first one it finds.

Let's say you have 2 resources in relationship: `Company` and `User`, where a company has multiple users. You can declare
the following routes:

- `/users`
- `/users/{id}`
- `/companies/{companyId}/users`
- `/companies/{companyId}/users/{id}`

The first routes (`/users...`) are only accessible by the admin, and the others by regular users. Calling
`/companies/{companyId}/users` should return IRIs matching `/companies/{companyId}/users/{id}` to not expose an admin
route to regular users.

To do so, use the `itemUriTemplate` option only available on `GetCollection` and `Post` operations:

<code-selector>

```php
<?php
// api/src/Entity/User.php
namespace App\Entity;

use ApiPlatform\Metadata\ApiResource;
use ApiPlatform\Metadata\Get;
use ApiPlatform\Metadata\GetCollection;
use ApiPlatform\Metadata\Post;

#[GetCollection] // auto-generated path will be /users
#[Get] // auto-generated path will be /users/{id}
#[GetCollection(uriTemplate: '/companies/{companyId}/users', itemUriTemplate: '/companies/{companyId}/users/{id}'/*, ... */)]
#[Post(uriTemplate: '/companies/{companyId}/users', itemUriTemplate: '/companies/{companyId}/users/{id}'/*, ... */)]
#[Get(uriTemplate: '/companies/{companyId}/users/{id}'/*, ... */)]
class User
{
    //...
}
```

```yaml
# api/config/api_platform/resources.yaml
resources:
    App\Entity\User:
        - operations:
            ApiPlatform\Metadata\GetCollection: ~
            ApiPlatform\Metadata\Get: ~
        - operations:
            ApiPlatform\Metadata\GetCollection:
                uriTemplate: /companies/{companyId}/users
                itemUriTemplate: /companies/{companyId}/users/{id}
                # ...
            ApiPlatform\Metadata\Post:
                uriTemplate: /companies/{companyId}/users
                itemUriTemplate: /companies/{companyId}/users/{id}
                # ...
            ApiPlatform\Metadata\Get:
                uriTemplate: /companies/{companyId}/users/{id}
                # ...
```

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<!-- api/config/api_platform/resources.xml -->

<resources xmlns="https://api-platform.com/schema/metadata/resources-3.0"
        xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:schemaLocation="https://api-platform.com/schema/metadata/resources-3.0
        https://api-platform.com/schema/metadata/resources-3.0.xsd">
    <resource class="App\Entity\User">
        <operations>
            <operation class="ApiPlatform\Metadata\GetCollection" />
            <operation class="ApiPlatform\Metadata\Get" />
        </operations>
    </resource>

    <resource class="App\Entity\User">
        <operations>
            <operation class="ApiPlatform\Metadata\GetCollection" uriTemplate="/companies/{companyId}/users" itemUriTemplate="/companies/{companyId}/users/{id}" />
            <operation class="ApiPlatform\Metadata\Post" uriTemplate="/companies/{companyId}/users" itemUriTemplate="/companies/{companyId}/users/{id}" />
            <operation class="ApiPlatform\Metadata\Get" uriTemplate="/companies/{companyId}/users/{id}" />
        </operations>
    </resource>
</resources>
```

</code-selector>

API Platform will find the operation matching this `itemUriTemplate` and use it to generate the IRI.

If this option is not set, the first `Get` operation is used to generate the IRI.

## Expose a Model Without Any Routes

Sometimes, you may want to expose a model, but want it to be used through subrequests only, and never through item or collection operations.
Because the OpenAPI standard requires at least one route to be exposed to make your models consumable, let's see how you can manage this kind
of issue.

Let's say you have the following entities in your project:

```php
<?php
// api/src/Entity/Place.php
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
// api/src/Entity/Weather.php
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
// api/src/Entity/Place.php
namespace App\Entity;

use ApiPlatform\Metadata\Get;
use ApiPlatform\Metadata\Put;
use ApiPlatform\Metadata\Delete;
use ApiPlatform\Metadata\GetCollection;
use ApiPlatform\Metadata\Post;
use ApiPlatform\Metadata\ApiResource;
use App\Controller\GetWeather;
use Doctrine\ORM\Mapping as ORM;

#[ApiResource(
    operations: [
        new Get(),
        new Put(),
        new Delete(),
        new Get(name: 'weather', uriTemplate: '/places/{id}/weather', controller: GetWeather::class),
        new GetCollection(),
        new Post(),
    ]
)]
#[ORM\Entity]
class Place
{
    // ...
```

The `GetWeather` controller fetches the weather for the given city and returns an instance of the `Weather` entity.
This implies that API Platform has to know about this entity, so we will need to make it an API resource too:

```php
<?php
// api/src/Entity/Weather.php
namespace App\Entity;

use ApiPlatform\Metadata\ApiResource;

#[ApiResource]
class Weather
{
    // ...
```

This will expose the `Weather` model, but also all the default CRUD routes: `GET`, `PUT`, `PATCH`, `DELETE` and `POST`, which is a non-sense in our context.
Since we are required to expose at least one route, let's expose just one:

```php
<?php
// api/src/Entity/Weather.php
namespace App\Entity;

use ApiPlatform\Metadata\ApiResource;
use ApiPlatform\Metadata\Get;

#[ApiResource(operations: [
    new Get(controller: SomeRandomController::class)
])]
class Weather
{
    // ...
}
```

This way, we expose a route that will do… nothing. Note that the controller does not even need to exist.

It's almost done, we have just one final issue: our fake item operation is visible in the API docs.
To remove it, we will need to [decorate the Swagger documentation](openapi.md#overriding-the-openapi-specification).
Then, remove the route from the decorator:

```php
<?php
// src/OpenApi/OpenApiFactory.php
namespace App\OpenApi;

use ApiPlatform\OpenApi\Factory\OpenApiFactoryInterface;
use ApiPlatform\OpenApi\OpenApi;
use ApiPlatform\OpenApi\Model;

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

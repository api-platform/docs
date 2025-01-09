# Creating Custom Operations and Controllers

Note: using custom controllers with API Platform is **discouraged**. Also, GraphQL is **not supported**.
[For most use cases, better extension points, working both with REST and GraphQL, are available](design.md).

API Platform can leverage the Symfony routing system to register custom operations related to custom controllers. Such custom
controllers can be any valid [Symfony controller](http://symfony.com/doc/current/book/controller.html), including standard
Symfony controllers extending the [`Symfony\Bundle\FrameworkBundle\Controller\AbstractController`](http://api.symfony.com/4.1/Symfony/Bundle/FrameworkBundle/Controller/AbstractController.html)
helper class.

However, API Platform recommends to use **action classes** instead of typical Symfony controllers. Internally, API Platform
implements the [Action-Domain-Responder](https://github.com/pmjones/adr) pattern (ADR), a web-specific refinement of
[MVC](https://en.wikipedia.org/wiki/Model%E2%80%93view%E2%80%93controller).

The distribution of API Platform also eases the implementation of the ADR pattern: it automatically registers action classes
stored in `src/Controller` as autowired services.

Thanks to the [autowiring](http://symfony.com/doc/current/components/dependency_injection/autowiring.html) feature of the
Symfony Dependency Injection container, services required by an action can be type-hinted in its constructor, it will be
automatically instantiated and injected, without having to declare it explicitly.

In the following examples, the built-in `GET` operation is registered as well as a custom operation called `post_publication`.

By default, API Platform uses the first `GET` operation defined in `itemOperations` to generate the IRI of an item and the first `GET` operation defined in `collectionOperations` to generate the IRI of a collection.

If you create a custom operation, you will probably want to properly document it.
See the [OpenAPI](swagger.md) part of the documentation to do so.

First, let's create your custom operation:

```php
<?php
// api/src/Controller/CreateBookPublication.php

namespace App\Controller;

use App\Entity\Book;
use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
use Symfony\Component\HttpKernel\Attribute\AsController;

#[AsController]
class CreateBookPublication extends AbstractController
{
    private $bookPublishingHandler;

    public function __construct(BookPublishingHandler $bookPublishingHandler)
    {
        $this->bookPublishingHandler = $bookPublishingHandler;
    }

    public function __invoke(Book $data): Book
    {
        $this->bookPublishingHandler->handle($data);

        return $data;
    }
}
```

This custom operation behaves exactly like the built-in operation: it returns a JSON-LD document corresponding to the id
passed in the URL.

Here we consider that [autowiring](https://symfony.com/doc/current/service_container/autowiring.html) is enabled for
controller classes (the default when using the API Platform distribution).
This action will be automatically registered as a service (the service name is the same as the class name:
`App\Controller\CreateBookPublication`).

API Platform automatically retrieves the appropriate PHP entity using the data provider then deserializes user data in it,
and for `POST`, `PUT` and `PATCH` requests updates the entity with data provided by the user.

**Warning: the `__invoke()` method parameter [MUST be called `$data`](https://symfony.com/doc/current/components/http_kernel.html#4-getting-the-controller-arguments)**, otherwise, it will not be filled correctly!

Services (`$bookPublishingHandler` here) are automatically injected thanks to the autowiring feature. You can type-hint any service
you need and it will be autowired too.

The `__invoke` method of the action is called when the matching route is hit. It can return either an instance of
`Symfony\Component\HttpFoundation\Response` (that will be displayed to the client immediately by the Symfony kernel) or,
like in this example, an instance of an entity mapped as a resource (or a collection of instances for collection operations).
In this case, the entity will pass through [all built-in event listeners](events.md#built-in-event-listeners) of API Platform. It will be
automatically validated, persisted and serialized in JSON-LD. Then the Symfony kernel will send the resulting document to
the client.

The routing has not been configured yet because we will add it at the resource configuration level:

<code-selector>

```php
<?php
// api/src/Entity/Book.php

use ApiPlatform\Core\Annotation\ApiResource;
use App\Controller\CreateBookPublication;

#[ApiResource(itemOperations: [
    'get',
    'post_publication' => [
        'method' => 'POST',
        'path' => '/books/{id}/publication',
        'controller' => CreateBookPublication::class,
    ],
])]
class Book
{
    // ...
}
```

```yaml
# config/api_platform/resources.yaml
App\Entity\Book:
    itemOperations:
        get: ~
        post_publication:
            method: POST
            path: /books/{id}/publication
            controller: App\Controller\CreateBookPublication
```

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<!-- api/config/api_platform/resources.xml -->

<resources
        xmlns="https://api-platform.com/schema/metadata"
        xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:schemaLocation="https://api-platform.com/schema/metadata
        https://api-platform.com/schema/metadata/metadata-2.0.xsd">
    <resource class="App\Entity\Book">
        <itemOperations>
            <itemOperation name="get" />
            <itemOperation name="post_publication">
                <attribute name="method">POST</attribute>
                <attribute name="path">/books/{id}/publication</attribute>
                <attribute name="controller">App\Controller\CreateBookPublication</attribute>
            </itemOperation>
        </itemOperations>
    </resource>
</resources>
```

</code-selector>

It is mandatory to set the `method`, `path` and `controller` attributes. They allow API Platform to configure the routing path and
the associated controller respectively.

## Using Serialization Groups

You may want different serialization groups for your custom operations. Just configure the proper `normalization_context` and/or `denormalization_context` in your operation:

<code-selector>

```php
<?php
// api/src/Entity/Book.php

use ApiPlatform\Core\Annotation\ApiResource;
use App\Controller\CreateBookPublication;
use Symfony\Component\Serializer\Annotation\Groups;

#[ApiResource(itemOperations: [
    'get',
    'post_publication' => [
        'method' => 'POST',
        'path' => '/books/{id}/publication',
        'controller' => CreateBookPublication::class,
        'normalization_context' => ['groups' => 'publication'],
    ],
])]
class Book
{
    // ...

    #[Groups(['publication'])]
    public $isbn;

    // ...
}
```

```yaml
# config/api_platform/resources.yaml
App\Entity\Book:
    itemOperations:
        get: ~
        post_publication:
            method: POST
            path: /books/{id}/publication
            controller: App\Controller\CreateBookPublication
            normalization_context:
                groups: ['publication']
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
                <attribute name="method">POST</attribute>
                <attribute name="path">/books/{id}/publication</attribute>
                <attribute name="controller">App\Controller\CreateBookPublication</attribute>
                <attribute name="normalization_context">
                    <attribute name="groups">
                        <attribute>publication</attribute>
                    </attribute>
                </attribute>
            </itemOperation>
        </itemOperations>
    </resource>
</resources>
```

</code-selector>

## Retrieving the Entity

If you want to bypass the automatic retrieval of the entity in your custom operation, you can set `"read"=false` in the
operation attribute:

<code-selector>

```php
<?php
// api/src/Entity/Book.php

use ApiPlatform\Core\Annotation\ApiResource;
use App\Controller\CreateBookPublication;

#[ApiResource(itemOperations: [
    'get',
    'post_publication' => [
        'method' => 'POST',
        'path' => '/books/{id}/publication',
        'controller' => CreateBookPublication::class,
        'read' => false,
    ],
])]
class Book
{
    // ...
}
```

```yaml
# config/api_platform/resources.yaml
App\Entity\Book:
    itemOperations:
        get: ~
        post_publication:
            method: POST
            path: /books/{id}/publication
            controller: App\Controller\CreateBookPublication
            read: false
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
                <attribute name="method">POST</attribute>
                <attribute name="path">/books/{id}/publication</attribute>
                <attribute name="controller">App\Controller\CreateBookPublication</attribute>
                <attribute name="read">false</attribute>
            </itemOperation>
        </itemOperations>
    </resource>
</resources>
```

</code-selector>

This way, it will skip the `ReadListener`. You can do the same for some other built-in listeners. See [Built-in Event Listeners](events.md#built-in-event-listeners)
for more information.

In your custom controller, the `__invoke()` method parameter should be called the same as the entity identifier.
So for the path `/user/{uuid}/bookmarks`, you must use `__invoke(string $uuid)`.

## Alternative Method

There is another way to create a custom operation. However, we do not encourage its use. Indeed, this one disperses
the configuration at the same time in the routing and the resource configuration.

The `post_publication` operation references the Symfony route named `book_post_publication`.

Since version 2.3, you can also use the route name as operation name by convention, as shown in the following example
for `book_post_discontinuation` when neither `method` nor `route_name` attributes are specified.

First, let's create your resource configuration:

<code-selector>

```php
<?php
// api/src/Entity/Book.php

use ApiPlatform\Core\Annotation\ApiResource;

#[ApiResource(itemOperations: [
    'get',
    'post_publication' => ['route_name' => 'book_post_publication'],
    'book_post_discontinuation',
])]
class Book
{
    // ...
}
```

```yaml
# config/api_platform/resources.yaml
App\Entity\Book:
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
        name: 'book_post_publication',
        path: '/books/{id}/publication',
        methods: ['POST'],
        defaults: [
            '_api_resource_class' => Book::class,
            '_api_item_operation_name' => 'post_publication',
        ],
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
the same thing as the previous example:

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
    public function createPublication(Book $data, BookPublishingHandler $bookPublishingHandler): Book
    {
        return $bookPublishingHandler->handle($data);
    }
}
```

```yaml
# config/routes.yaml
book_post_publication:
    path: /books/{id}/publication
    methods: ['POST']
    defaults:
        _controller: App\Controller\BookController::createPublication
        _api_resource_class: App\Entity\Book
        _api_item_operation_name: post_publication
```

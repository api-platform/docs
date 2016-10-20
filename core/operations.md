# Operations

API Platform Core relies on the concept of operations. Operations can be applied to a resource exposed by the API. From
an implementation point of view, an operation is a link between a resource, a route and its related controller.

API Platform automatically registers typical [CRUD](https://en.wikipedia.org/wiki/Create,_read,_update_and_delete) operations
and describes them in the exposed documentation (Hydra and NelmioApiDoc). It also creates and registers routes corresponding
to these operations in the Symfony routing system (if it is available).

The behavior of built-in operations is briefly presented in the [Getting started](getting-started.md#mapping-the-entities)
guide.

The list of enabled operations can be configured on a per resource basis. Creating custom operations on specific routes
is also possible.

There are two types of operations: collection operations and item operations.

Collection operations act on a collection of resources. By default two routes are implemented: `POST` and `GET`. Item
operations act on an individual resource. 3 default routes are defined `GET`, `PUT` and `DELETE`.

When the `ApiPlatform\Core\Annotation\ApiResource` annotation is applied to an entity class, the following built-in CRUD
operations are automatically enabled:

*Collection operations*

Method | Mandatory | Description
-------|-----------|------------------------------------------
`GET`  | yes       | Retrieve the (paginated) list of elements
`POST` | no        | Create a new element

*Item operations*

Method   | Mandatory | Description
---------|-----------|------------------
`GET`    | yes       | Retrieve element
`PUT`    | no        | Update an element
`DELETE` | no        | Delete an element

## Enabling and Disabling Operations

If no operation is specified, all default CRUD operations are automatically registered. It is also possible - and recommended
for large projects - to define operations explicitly.

Keep in mind that `collectionOperations` and `itemOperations` behave independently. For instance, if you don't explicitly
configure operations for `collectionOperations`, `GET` and `POST` operations will be automatically registered, even if you
explicitly configure `itemOperations`. The reverse is also true.

Operations can be configured using annotations, XML or YAML. In the following examples, we enable only the built-in operation
for the `GET` method for both `collectionOperations` and `itemOperations` to create a readonly endpoint.

`itemOperations` and `collectionOperations` are arrays containing a list of operation. Each operation is defined by a key
corresponding to the name of the operation that can be anything you want and an array of properties as value.

<configurations>

```php
<?php

// src/AppBundle/Entity/Book.php

use ApiPlatform\Core\Annotation\ApiResource;

/**
 * @ApiResource(collectionOperations={"get"={"method"="GET"}}, itemOperations={"get"={"method"="GET"}})
 */
class Book
{
    // ...
}
```

```yaml
# src/AppBundle/Resources/config/api_resources.yml
product:
    class: 'AppBundle\Entity\Book'
    collectionOperations:
        get:
            method: 'GET' # nothing more to add if we want to keep the default controller
    itemOperations:
        get:
            method: 'GET'
```

```xml
<!-- src/AppBundle/Resources/config/api_resources.xml -->
<?xml version="1.0" encoding="UTF-8" ?>
<resources>
    <resource class="AppBundle\Entity\Book">
        <itemOperations type="collection">
            <operation key="get" method="GET" />
        </itemOperations>
        <collectionOperations type="collection">
            <operation key="get" method="GET" />
        </collectionOperations>
    </resource>
</resources>
```

</configurations>

API Platform Core is smart enough to automatically register the applicable Symfony route referencing a built-in CRUD action
just by specifying the enabled HTTP method.

## Configuring Operations

The URL, the HTTP method and the Hydra context passed to documentation generators of operations is easy to configure.

In the next example, both `GET` and `PUT` operations are registered with custom URLs. Those will override the default generated
URLs. In addition to that, we replace the Hydra context for the `PUT` operation.

<configurations>

```php
<?php

// src/AppBundle/Entity/Book.php

use ApiPlatform\Core\Annotation\ApiResource;

/**
 * @ApiResource(itemOperations={
 *     "get"={"method"="GET", "path"="/grimoire/{id}"},
 *     "put"={"method"="PUT", "path"="/grimoire/{id}/update", "hydra_context"={"foo"="bar"}},
 * })
 */
class Book
{
    //...
}
```

```yaml
# src/AppBundle/Resources/config/resources.yml
product:
    class: 'AppBundle\Entity\Book'
    itemOperations:
        get:
            method: 'GET'
            path: '/grimoire/{id}'
        put:
            method: 'PUT'
            path: '/grimoire/{id}/update'
            hydra_context: { foo: 'bar' }
```

```xml
<!-- src/Acme/BlogBundle/Resources/config/resources.xml -->
<?xml version="1.0" encoding="UTF-8" ?>
<resources>
    <resource class="AppBundle\Entity\Book">
        <itemOperations type="collection">
            <operation key="get" method="GET" />
            <operation key="put" method="GET" path="/grimoire/{id}/update">
                <attribute key="hydra_context" type="collection">
                    <attribute key="foo">bar</attribute>
                </attribute>
            </operation>
        </itemOperations>
    </resource>
</resources>
```

</configurations>

## Creating Custom Operations and Controllers

API Platform can leverage the Symfony routing system to register custom operation related to custom controllers. Such custom
controllers can be any valid [Symfony controller](http://symfony.com/doc/current/book/controller.html), including standard
Symfony controllers extending the [`Symfony\Bundle\FrameworkBundle\Controller\Controller`](http://api.symfony.com/3.1/Symfony/Bundle/FrameworkBundle/Controller/Controller.html)
helper class.

However, API Platform recommends to use **action classes** instead of typical Symfony controllers. Internally, API Platform
implements the [Action-Domain-Responder](https://github.com/pmjones/adr) pattern (ADR), a web-specific refinement of [MVC](https://en.wikipedia.org/wiki/Model%E2%80%93view%E2%80%93controller).

Note: [the event system](events.md) should be preferred over custom controllers when applicable.

The distribution of API Platform also comes with an enhanced action system for Symfony pre-installed: [DunglasActionBundle](https://github.com/dunglas/DunglasActionBundle).
*DunglasActionBundle* eases the implementation of the ADR pattern with Symfony and improves the developer experience.

It automatically registers action classes stored in `src/*Bundle/Action` and `src/*Bundle/Controller` directories as autowired
services.

Thanks to the [autowiring](http://symfony.com/doc/current/components/dependency_injection/autowiring.html) feature of the
Symfony dependency injection container, services required by an action can be type-hinted in its controller, it will be automatically
instantiated and injected, without having to declare it explicitly.

In the following example, the built-in `GET` operation is registered as well as a custom operation called `special`.
The `special` operation reference the Symfony route named `book_special`.

<configurations>

```php
<?php

// src/AppBundle/Entity/Book.php

use ApiPlatform\Core\Annotation\ApiResource;

/**
 * @ApiResource(itemOperations={
 *     "get"={"method"="GET"},
 *     "special"={"route_name"="book_special"}
 * })
 */
class Book
{
    //...
}
```

```yaml
# src/AppBundle/Resources/config/resources.yml
product:
    class: 'AppBundle\Entity\Book'
    itemOperations:
        get:
            method: 'GET'
        special:
            route_name: 'book_special'
```

```xml
<!-- src/Acme/BlogBundle/Resources/config/resources.xml -->
<?xml version="1.0" encoding="UTF-8" ?>
<resources>
    <resource class="AppBundle\Entity\Book">
        <itemOperations type="collection">
            <operation key="get" method="GET" />
            <operation key="special" routeName="book_special" />
        </itemOperations>
    </resource>
</resources>
```

</configurations>

API Platform will automatically map this `special` operation with the route `book_special`. Let's create a custom action
and its related route using annotations:

```php
<?php

// src/AppBundle/Action/BookSpecial.php

namespace AppBundle\Action;

use AppBundle\Entity\Book;
use Doctrine\Common\Persistence\ManagerRegistry;
use Sensio\Bundle\FrameworkExtraBundle\Configuration\Method;
use Symfony\Component\Routing\Annotation\Route;

class BookSpecial
{
    private $myService;

    public function __construct(MyService $myService)
    {
        $this->myService = $myService;
    }

    /**
     * @Route(
     *     name="book_special",
     *     path="/books/{id}/special",
     *     defaults={"_api_resource_class"=Book::class, "_api_item_operation_name"="special"}
     * )
     * @Method("PUT")
     */
    public function __invoke($data) // API Platform retrieves the PHP entity using the data provider then (for POST and
                                    // PUT method) deserializes user data in it. Then passes it to the action. Here $data
                                    // is an instance of Book having the given ID. By convention, the action's parameter
                                    // must be called $data.
    {
        $this->myService->doSomething($data);

        return $data; // API Platform will automatically validate, persist (if you use Doctrine) and serialize an entity
                      // for you. If you prefer to do it yourself, return an instance of Symfony\Component\HttpFoundation\Response
    }
}
```

This custom operation behaves exactly like the built-in operation: it returns a JSON-LD document corresponding to the id
passed in the URL.

It is mandatory to set the `_api_resource_class` and `_api_item_operation_name` (or `_api_collection_operation_name` for a collection
operation) in the parameters of the route (`defaults` key). It allows API Platform and the Symfony routing system to hook
together.

Here we consider that DunglasActionBundle is installed (the default when using the API Platform distribution). This
action will be automatically registered as a service (the service name is the same as the class name: `AppBundle\Action\BookSpecial`).

API Platform automatically retrieve the appropriate PHP entity then then deserializes it, and for `POST` and `PUT` requests
updates the entity with data provided by the user.

Services (`$myService` here) are automatically injected thanks to the autowiring feature. You can type-hint any service
you need and it will be autowired too.

The `__invoke` method of the action is called when the matching route is hit. It can return either an instance of `Symfony\Component\HttpFoundation\Response`
(that will be displayed to the client immediately by the Symfony kernel) or, like in this example, an instance of an entity
mapped as a resource (or a collection of instances for collection operations).
In this case, the entity will pass through [all built-in event listeners](the-event-system.md) of API Platform. It will be
automatically validated, persisted and serialized in JSON-LD. Then the Symfony kernel will send the resulting document to
the client.

Alternatively, you can also use standard Symfony controller and YAML or XML route declarations. The following example do
exactly the same thing than the previous example in a more Symfony-like fashion:

```php
<?php

// src/AppBundle/Controller/BookController.php

namespace AppBundle\Controller;

use AppBundle\Entity\Book;
use Symfony\Bundle\FrameworkBundle\Controller\Controller;

class BookController extends Controller
{
    public function specialAction($data)
    {
        return $this->get('my_service')->doSomething(Book::class, $id);
    }
}
```

```yaml
# app/config/routing.yml

book_special:
    path: '/books/{id}/special'
    methods:  ['PUT']
    defaults:
        _controller: 'AppBundle:Book:special'
        _api_resource_class: 'AppBundle\Entity\Book'
        _api_item_operation_name: 'special'
```

Previous chapter: [NelmioApiDocBundle integration](nelmio-api-doc.md)

Next chapter: [Filters](filters.md)

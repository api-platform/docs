# Operations

API Platform Core relies on the concept of operations. Operations can be applied to a resource exposed by the API. From
an implementation point of view, an operation is a link between a resource, a route and its related controller.

API Platform automatically registers typical [CRUD](https://en.wikipedia.org/wiki/Create,_read,_update_and_delete) operations
and describes them in the exposed documentation (Hydra and Swagger). It also creates and registers routes corresponding
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
corresponding to the name of the operation that can be anything you want and an array of properties as value. If an
empty list of operations is provided, all operations are disabled.

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
# src/AppBundle/Resources/config/api_resources/resources.yml

AppBundle\Entity\Book:
    collectionOperations:
        get:
            method: 'GET' # nothing more to add if we want to keep the default controller
    itemOperations:
        get:
            method: 'GET'
```

```xml
<!-- src/AppBundle/Resources/config/api_resources/resources.xml -->

<?xml version="1.0" encoding="UTF-8" ?>
<resources xmlns="https://api-platform.com/schema/metadata"
           xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
           xsi:schemaLocation="https://api-platform.com/schema/metadata
           https://api-platform.com/schema/metadata/metadata-2.0.xsd">
    <resource class="AppBundle\Entity\Book">
        <itemOperations>
            <itemOperation name="get">
                <attribute name="method">GET</attribute>
            </itemOperation>
        </itemOperations>
        <collectionOperations>
            <collectionOperation name="get">
                <attribute name="method">GET</attribute>
            </collectionOperation>
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
 *     "put"={"method"="PUT", "path"="/grimoire/{id}/update", "hydra_context"={"foo"="bar"},
 * })
 */
class Book
{
    //...
}
```

```yaml
# src/AppBundle/Resources/config/api_resources/resources.yml

AppBundle\Entity\Book:
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
<!-- src/AppBundle/Resources/config/api_resources/resources.xml -->

<?xml version="1.0" encoding="UTF-8" ?>
<resources xmlns="https://api-platform.com/schema/metadata"
           xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
           xsi:schemaLocation="https://api-platform.com/schema/metadata
           https://api-platform.com/schema/metadata/metadata-2.0.xsd">
    <resource class="AppBundle\Entity\Book">
        <itemOperations>
            <itemOperation name="get">
                <attribute name="method">GET</attribute>
                <attribute name="path">/grimoire/{id}</attribute>
            </itemOperation>
            <itemOperation name="put">
                <attribute name="method">PUT</attribute>
                <attribute name="path">/grimoire/{id}/update</attribute>
                <attribute name="hydra_context">
                    <attribute name="foo">bar</attribute>
                </attribute>
            </itemOperation>
        </itemOperations>
    </resource>
</resources>
```

</configurations>

## Subresources

Since ApiPlatform 2.1, you can declare subresources. A subresource is a collection or an item that belongs to another resource. The starting point of a subresource must be a relation on an existing resource.

For example, let's create two entities (Question, Answer) and set up a subresource so that `/question/42/answer` gives us the answer to the question 42:

```php

// src/AppBundle/Entity/Answer.php

use ApiPlatform\Core\Annotation\ApiResource;
use Doctrine\ORM\Mapping as ORM;

/**
 * Answer.
 *
 * @ORM\Entity
 * @ApiResource
 */
class Answer
{
    /**
     * @ORM\Column(type="integer")
     * @ORM\Id
     * @ORM\GeneratedValue(strategy="AUTO")
     */
    private $id;

    /**
     * @ORM\Column
     */
    public $content;

    /**
     * @ORM\OneToOne(targetEntity="Question", mappedBy="answer")
     */
    public $question;

    public function getId()
    {
      return $this->id;
    }
}
```

```php
<?php

// src/AppBundle/Entity/Question.php

use ApiPlatform\Core\Annotation\ApiProperty;
use ApiPlatform\Core\Annotation\ApiResource;
use ApiPlatform\Core\Annotation\ApiSubresource;
use Doctrine\ORM\Mapping as ORM;

/**
 * @ORM\Entity
 * @ApiResource
 */
class Question
{
    /**
     * @ORM\Column(type="integer")
     * @ORM\Id
     * @ORM\GeneratedValue(strategy="AUTO")
     */
    private $id;

    /**
     * @ORM\Column
     */
    public $content;

    /**
     * @ORM\OneToOne(targetEntity="Answer", inversedBy="question")
     * @ORM\JoinColumn(referencedColumnName="id", unique=true)
     * @ApiSubresource
     */
    public $answer;

    public function getId()
    {
      return $this->id;
    }
}
```

Note that all we had to do is to set up `@ApiSubresource` on the `Question::answer` relation. Because the `answer` is a to-one relation, we know that this subresource is an item. Therefore the response will look like this:

```json
{
  "@context": "/contexts/Answer",
  "@id": "/answers/42",
  "@type": "Answer",
  "id": 42,
  "content": "Life, the Universe, and Everything",
  "question": "/questions/42"
}
```

If you put the subresource on a relation that is to-many, you will retrieve a collection.

Last but not least, Subresources can be nested, such that `/questions/42/answer/comments` will get the collection of comments for the answer to question 42.

You may want custom groups on subresources. Because a subresource is nothing more than a collection operation, you can set `normalization_context` or `denormalization_context` on that operation. To do so, you need to override `collectionOperations`. Based on the above operation, because we retrieve an answer, we need to alter it's configuration:

<configurations>

```php
<?php

// src/AppBundle/Entity/Answer.php

use ApiPlatform\Core\Annotation\ApiResource;

/**
 * @ApiResource(collectionOperations={"api_questions_answer_get_subresource"={"method"="GET", "normalization_context"={"groups"={"foobar"}}}})
 */
class Answer
{
    // ...
}
```

```yaml
# src/AppBundle/Resources/config/api_resources/resources.yml

AppBundle\Entity\Answer:
    collectionOperations:
        api_questions_answer_get_subresource:
            method: 'GET' # nothing more to add if we want to keep the default controller
            normalization_context: {'groups': ['foobar']}
```

```xml
<!-- src/AppBundle/Resources/config/api_resources/resources.xml -->

<?xml version="1.0" encoding="UTF-8" ?>
<resources xmlns="https://api-platform.com/schema/metadata"
           xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
           xsi:schemaLocation="https://api-platform.com/schema/metadata
           https://api-platform.com/schema/metadata/metadata-2.0.xsd">
    <resource class="AppBundle\Entity\Answer">
        <collectionOperations>
            <collectionOperation name="api_questions_answer_get_subresource">
                <attribute name="method">GET</attribute>
                <attribute name="normalization_context">
                  <attribute name="groups">
                    <group>foobar</group>
                  </attribute>
                </attribute>
            </collectionOperation>
        </collectionOperations>
    </resource>
</resources>
```

</configurations>

Note that the operation name, here `api_questions_answer_get_subresource`, is the important keyword. It'll be automatically set to `$resources_$subresource(s)_get_subresource`. To find the correct operation name you may use `bin/console debug:router`.

## Creating Custom Operations and Controllers

API Platform can leverage the Symfony routing system to register custom operation related to custom controllers. Such custom
controllers can be any valid [Symfony controller](http://symfony.com/doc/current/book/controller.html), including standard
Symfony controllers extending the [`Symfony\Bundle\FrameworkBundle\Controller\Controller`](http://api.symfony.com/3.1/Symfony/Bundle/FrameworkBundle/Controller/Controller.html)
helper class.

However, API Platform recommends to use **action classes** instead of typical Symfony controllers. Internally, API Platform
implements the [Action-Domain-Responder](https://github.com/pmjones/adr) pattern (ADR), a web-specific refinement of [MVC](https://en.wikipedia.org/wiki/Model%E2%80%93view%E2%80%93controller).

Note: [the event system](events.md) should be preferred over custom controllers when applicable.

The distribution of API Platform also eases the implementation of the ADR pattern: it automatically registers action classes
stored in `src/AppBundle/Action` and `src/AppBundle/Controller` as autowired services.

Thanks to the [autowiring](http://symfony.com/doc/current/components/dependency_injection/autowiring.html) feature of the
Symfony Dependency Injection container, services required by an action can be type-hinted in its constructor, it will be
automatically instantiated and injected, without having to declare it explicitly.

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
# src/AppBundle/Resources/config/api_resources/resources.yml

AppBundle\Entity\Book:
    itemOperations:
        get:
            method: 'GET'
        special:
            route_name: 'book_special'
```

```xml
<!-- src/AppBundle/Resources/config/api_resources/resources.xml -->

<?xml version="1.0" encoding="UTF-8" ?>
<resources xmlns="https://api-platform.com/schema/metadata"
           xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
           xsi:schemaLocation="https://api-platform.com/schema/metadata
           https://api-platform.com/schema/metadata/metadata-2.0.xsd">
    <resource class="AppBundle\Entity\Book">
        <itemOperations>
            <itemOperation name="get">
                <attribute name="method">GET</attribute>
            </itemOperation>
            <itemOperation name="special">
                <attribute name="route_name">book_special</attribute>
            </itemOperation>
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

Here we consider that the autowiring enabled for controller classes (the default when using the API Platform distribution).
This action will be automatically registered as a service (the service name is the same as the class name: `AppBundle\Action\BookSpecial`).

API Platform automatically retrieves the appropriate PHP entity then deserializes it, and for `POST` and `PUT` requests
updates the entity with data provided by the user.

Services (`$myService` here) are automatically injected thanks to the autowiring feature. You can type-hint any service
you need and it will be autowired too.

The `__invoke` method of the action is called when the matching route is hit. It can return either an instance of `Symfony\Component\HttpFoundation\Response`
(that will be displayed to the client immediately by the Symfony kernel) or, like in this example, an instance of an entity
mapped as a resource (or a collection of instances for collection operations).
In this case, the entity will pass through [all built-in event listeners](events.md) of API Platform. It will be
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
        return $this->get('my_service')->doSomething($data);
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

Previous chapter: [Configuration](configuration.md)

Next chapter: [Overriding Default Order](default-order.md)


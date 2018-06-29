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
operations act on an individual resource. 3 default routes are defined `GET`, `PUT` and `DELETE` (`PATCH` is also supported
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
---------|-----------|------------------
`GET`    | yes       | Retrieve element
`PUT`    | no        | Update an element
`DELETE` | no        | Delete an element

## Enabling and Disabling Operations

If no operation are specified, all default CRUD operations are automatically registered. It is also possible - and recommended
for large projects - to define operations explicitly.

Keep in mind that `collectionOperations` and `itemOperations` behave independently. For instance, if you don't explicitly
configure operations for `collectionOperations`, `GET` and `POST` operations will be automatically registered, even if you
explicitly configure `itemOperations`. The reverse is also true.

Operations can be configured using annotations, XML or YAML. In the following examples, we enable only the built-in operation
for the `GET` method for both `collectionOperations` and `itemOperations` to create a readonly endpoint.

`itemOperations` and `collectionOperations` are arrays containing a list of operation. Each operation is defined by a key
corresponding to the name of the operation that can be anything you want and an array of properties as value. If an
empty list of operations is provided, all operations are disabled.

If the operation's name match a supported HTTP methods (`GET`, `POST`, `PUT` or `DELETE`), the corresponding `method` property
will be automatically added.

```php
<?php
// api/src/Entity/Book.php

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

## Configuring Operations

The URL, the HTTP method and the Hydra context passed to documentation generators of operations is easy to configure.

In the next example, both `GET` and `PUT` operations are registered with custom URLs. Those will override the default generated
URLs. In addition to that, we replace the Hydra context for the `PUT` operation, and require the `id` parameter to be an integer.

```php
<?php
// api/src/Entity/Book.php

use ApiPlatform\Core\Annotation\ApiResource;

/**
 * ...
 * @ApiResource(itemOperations={
 *     "get"={"method"="GET", "path"="/grimoire/{id}", "requirements"={"id"="\d+"}, "defaults"={"color"="brown"}, "options"={"my_option"="my_option_value", "schemes"={"https"}, "host"="{subdomain}.api-platform.com"}},
 *     "put"={"method"="PUT", "path"="/grimoire/{id}/update", "hydra_context"={"foo"="bar"}},
 * })
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
        put:
            method: 'PUT'
            path: '/grimoire/{id}/update'
            hydra_context: { foo: 'bar' }
            requirements:
                id: '\d+'
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
        <itemOperations>
            <itemOperation name="get">
                <attribute name="method">GET</attribute>
                <attribute name="path">/grimoire/{id}</attribute>
                <attribute name="requirements">
                    <attribute name="id">\d+</attribute>
                </attribute>
                <attribute name="defaults">
                    <attribute name="color">brown</attribute>
                </attribute>
                <attribute name="options">
                    <attribute name="color">brown</attribute>
                </attribute>
                <attribute name="schemes">
                    <attribute>https</attribute>
                </attribute>
                <attribute name="host">{subdomain}.api-platform.com</attribute>
            </itemOperation>
            <itemOperation name="put">
                <attribute name="method">PUT</attribute>
                <attribute name="path">/grimoire/{id}/update</attribute>
                <attribute name="hydra_context">
                    <attribute name="foo">bar</attribute>
                </attribute>
                <attribute name="requirements">
                    <attribute name="id">\d+</attribute>
                </attribute>
            </itemOperation>
        </itemOperations>
    </resource>
</resources>
```

In all the previous examples, you can safely remove the `method` because the method name always match the operation name.

## Subresources

Since ApiPlatform 2.1, you can declare subresources. A subresource is a collection or an item that belongs to another resource.
The starting point of a subresource must be a relation on an existing resource.

For example, let's create two entities (Question, Answer) and set up a subresource so that `/question/42/answer` gives us
the answer to the question 42:

```php
<?php
// api/src/Entity/Answer.php

namespace App\Entity;

use ApiPlatform\Core\Annotation\ApiResource;
use Doctrine\ORM\Mapping as ORM;

/**
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

    public function getId(): ?int
    {
        return $this->id;
    }
}
```

```php
<?php
// api/src/Entity/Question.php

namespace App\Entity;

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

    public function getId(): ?int
    {
        return $this->id;
    }
}
```

Alternatively, you can use the YAML configuration format:

```yaml
# api/config/api_platform/resources.yaml
App\Entity\Answer: ~
App\Entity\Question:
    properties:
        answer:
            subresource:
                resourceClass: 'App\Entity\Answer'
                collection: false
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

```php
<?php
// api/src/Entity/Answer.php

namespace App\Entity;

use ApiPlatform\Core\Annotation\ApiResource;

/**
 * @ApiResource(collectionOperations={"api_questions_answer_get_subresource"={"method"="GET", "normalization_context"={"groups"={"foobar"}}}})
 */
class Answer
{
    // ...
}
```

Or using YAML:

```yaml
# api/config/api_platform/resources.yaml
App\Entity\Answer:
    collectionOperations:
        api_questions_answer_get_subresource:
            method: 'GET' # nothing more to add if we want to keep the default controller
            normalization_context: {'groups': ['foobar']}
```

Or in XML:

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<!-- api/config/api_platform/resources.xml -->

<resources xmlns="https://api-platform.com/schema/metadata"
           xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
           xsi:schemaLocation="https://api-platform.com/schema/metadata
           https://api-platform.com/schema/metadata/metadata-2.0.xsd">
    <resource class="App\Entity\Answer">
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

In the previous examples, the `method` attribute is mandatory, because the operation name doesn't match a supported HTTP
method.

Note that the operation name, here `api_questions_answer_get_subresource`, is the important keyword.
It'll be automatically set to `$resources_$subresource(s)_get_subresource`. To find the correct operation name you
may use `bin/console debug:router`.

### Control the Path of Subresources

You can control the path of subresources with the `path` option of the `subresourceOperations` parameter:

```php
<?php
// api/src/Entity/Question.php

/**
 * ...
 * @ApiResource(
 *      subresourceOperations={
 *          "answer_get_subresource"= {
 *              "method"="GET",
 *              "path"="/questions/{id}/all-answers",
 *          },
 *      },
 * )
 */
class Question
{
}
```

### Control the Depth of Subresources

You can control depth of subresources with the parameter `maxDepth`. For example, if `Answer` entity also have subresource
such as `comments`and you don't want the route `api/questions/{id}/answers/{id}/comments` to be generated. You can do this by adding the parameter maxDepth in ApiSubresource annotation or YAML/XML file configuration.

```php
<?php
// api/src/Entity/Question.php

use ApiPlatform\Core\Annotation\ApiProperty;
use ApiPlatform\Core\Annotation\ApiResource;
use ApiPlatform\Core\Annotation\ApiSubresource;

/**
 * ...
 * @ApiResource
 */
class Question
{
    /**
     * ...
     * @ApiSubresource(maxDepth=1)
     */
    public $answer;

    // ...
}
```

## Creating Custom Operations and Controllers

API Platform can leverage the Symfony routing system to register custom operations related to custom controllers. Such custom
controllers can be any valid [Symfony controller](http://symfony.com/doc/current/book/controller.html), including standard
Symfony controllers extending the [`Symfony\Bundle\FrameworkBundle\Controller\Controller`](http://api.symfony.com/3.1/Symfony/Bundle/FrameworkBundle/Controller/Controller.html)
helper class.

However, API Platform recommends to use **action classes** instead of typical Symfony controllers. Internally, API Platform
implements the [Action-Domain-Responder](https://github.com/pmjones/adr) pattern (ADR), a web-specific refinement of
[MVC](https://en.wikipedia.org/wiki/Model%E2%80%93view%E2%80%93controller).

Note: [the event system](events.md) should be preferred over custom controllers when applicable.

The distribution of API Platform also eases the implementation of the ADR pattern: it automatically registers action classes
stored in `src/App/Controller` as autowired services.

Thanks to the [autowiring](http://symfony.com/doc/current/components/dependency_injection/autowiring.html) feature of the
Symfony Dependency Injection container, services required by an action can be type-hinted in its constructor, it will be
automatically instantiated and injected, without having to declare it explicitly.

In the following examples, the built-in `GET` operation is registered as well as a custom operation called `special`.
The `special` operation reference the Symfony route named `book_special`.

Note: By default, API Platform uses the first `GET` operation defined in `itemOperations` to generate the IRI of an item and the first `GET` operation defined in `collectionOperations` to generate the IRI of a collection.

Note: With custom operation, you will probably want to properly document it. See the [swagger](swagger.md) part of the documentation to do so.

### Recommended Method

First, let's create your custom operation:

```php
<?php
// api/src/Controller/BookSpecial.php

namespace App\Controller;

use App\Entity\Book;

class BookSpecial
{
    private $myService;

    public function __construct(MyService $myService)
    {
        $this->myService = $myService;
    }

    public function __invoke(Book $data): Book
    {
        $this->myService->doSomething($data);

        return $data;
    }
}
```

This custom operation behaves exactly like the built-in operation: it returns a JSON-LD document corresponding to the id
passed in the URL.

Here we consider that [autowiring](https://symfony.com/doc/current/service_container/autowiring.html) is enabled for
controller classes (the default when using the API Platform distribution).
This action will be automatically registered as a service (the service name is the same as the class name:
`App\Controller\BookSpecial`).

API Platform automatically retrieves the appropriate PHP entity using the data provider then deserializes user data in it,
and for `POST` and `PUT` requests updates the entity with data provided by the user.
By convention, the action's parameter must be called `$data`.

Services (`$myService` here) are automatically injected thanks to the autowiring feature. You can type-hint any service
you need and it will be autowired too.

The `__invoke` method of the action is called when the matching route is hit. It can return either an instance of
`Symfony\Component\HttpFoundation\Response` (that will be displayed to the client immediately by the Symfony kernel) or,
like in this example, an instance of an entity mapped as a resource (or a collection of instances for collection operations).
In this case, the entity will pass through [all built-in event listeners](events.md) of API Platform. It will be
automatically validated, persisted and serialized in JSON-LD. Then the Symfony kernel will send the resulting document to
the client.

The routing has not been configured yet because we will add it at the resource configuration level:

```php
<?php
// src/Entity/Book.php

use ApiPlatform\Core\Annotation\ApiResource;
use App\Controller\BookSpecial;

/**
 * @ApiResource(itemOperations={
 *     "get",
 *     "special"={
 *         "method"="GET",
 *         "path"="/books/{id}/special",
 *         "controller"=BookSpecial::class
 *     }
 * })
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
    itemOperations:
        get: ~
        special:
            method: 'GET'
            path: '/books/{id}/special'
            controller: 'App\Controller\BookSpecial'
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
        <itemOperations>
            <itemOperation name="get" />
            <itemOperation name="special">
                <attribute name="method">GET</attribute>
                <attribute name="path">/books/{id}/special</attribute>
                <attribute name="controller">App\Controller\BookSpecial</attribute>
            </itemOperation>
        </itemOperations>
    </resource>
</resources>
```

It is mandatory to set the `method`, `path` and `controller` attributes. They allow API platform to configure the routing path and
the associated controller respectively.

If you want to bypass the automatic retrieval of the entity in your custom operation, you can set the parameter
`_api_receive` to `false` in the `default` attribute:

```php
<?php
// src/Entity/Book.php

use ApiPlatform\Core\Annotation\ApiResource;
use App\Controller\BookSpecial;

/**
 * @ApiResource(itemOperations={
 *     "get",
 *     "special"={
 *         "path"="/books/{id}/special",
 *         "controller"=BookSpecial::class,
 *         "defaults"={"_api_receive"=false}
 *     }
 * })
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
    itemOperations:
        get: ~
        special:
            path: '/books/{id}/special'
            controller: 'App\Controller\BookSpecial'
            defaults:
                _api_receive: false
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
        <itemOperations>
            <itemOperation name="get" />
            <itemOperation name="special">
                <attribute name="path">/books/{id}/special</attribute>
                <attribute name="controller">App\Controller\BookSpecial</attribute>
                <attribute name="defaults">
                    <attribute name="_api_receive">false</attribute>
                </attribute>
            </itemOperation>
        </itemOperations>
    </resource>
</resources>
```

This way, it will skip the `Read`, `Deserialize` and `Validate` listeners (see [the event system](events.md) for more
information).

### Alternative Method

There is another way to create a custom operation. However, we do not encourage its use. Indeed, this one disperses
the configuration at the same time in the routing and the resource configuration.

First, let's create your resource configuration:

```php
<?php
// src/Entity/Book.php

use ApiPlatform\Core\Annotation\ApiResource;

/**
 * @ApiResource(itemOperations={
 *     "get",
 *     "special"={"route_name"="book_special"}
 * })
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
    itemOperations:
        get: ~
        special:
            route_name: 'book_special'
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
        <itemOperations>
            <itemOperation name="get" />
            <itemOperation name="special">
                <attribute name="route_name">book_special</attribute>
            </itemOperation>
        </itemOperations>
    </resource>
</resources>
```

API Platform will automatically map this `special` operation with the route `book_special`. Let's create a custom action
and its related route using annotations:

```php
<?php
// api/src/Controller/BookSpecial.php

namespace App\Controller;

use App\Entity\Book;
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
     *     methods={"PUT"},
     *     defaults={
     *         "_api_resource_class"=Book::class,
     *         "_api_item_operation_name"="special"
     *     }
     * )
     */
    public function __invoke(Book $data): Book
    {
        $this->myService->doSomething($data);

        return $data;
    }
}
```

It is mandatory to set `_api_resource_class` and `_api_item_operation_name` (or `_api_collection_operation_name` for a collection
operation) in the parameters of the route (`defaults` key). It allows API Platform and the Symfony routing system to hook
together.

Alternatively, you can also use standard Symfony controller and YAML or XML route declarations. The following example does
exactly the same thing than the previous example in a more Symfony-like fashion:

```php
<?php
// api/src/Controller/BookController.php

namespace App\Controller;

use App\Entity\Book;
use Symfony\Bundle\FrameworkBundle\Controller\Controller;

class BookController extends Controller
{
    public function specialAction(Book $data, MyService $service): Book
    {
        return $service->doSomething($data);
    }
}
```

```yaml
# api/config/routes.yaml
book_special:
    path: '/books/{id}/special'
    methods:  ['PUT']
    defaults:
        _controller: '\App\Controller\Book::special'
        _api_resource_class: 'App\Entity\Book'
        _api_item_operation_name: 'special'
```

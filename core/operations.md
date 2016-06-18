# Operations

API Platform Core relies on the concept of operation. Operations can be applied to a resource exposed by the API. From a
technical point of view, an operation is a link between a resource an a route and its related controller.

API Platform is smart enough to automatically register typical [CRUD](https://en.wikipedia.org/wiki/Create,_read,_update_and_delete)
and to describe them in exposed documentations (Hydra and NelmioApiDoc). It also creates and registers route corresponding
to those operations in the Symfony routing system (if it is available).

The behavior of builtin operations is briefly presented in the [Getting started](getting-started.md#mapping-the-entities)
guide.

The list of enabled operations can be configured on a per resource basis. It also possible to create custom operations related
to specific routes.

There are two kind of operations: `collectionOperations` and `itemOperations`.

`collectionOperations` act on the collection of resources. By default two routes are implemented: `POST` and `GET`. On the
other hand, `itemOperations` act on an individual resource. 3 default routes are defined `GET`, `PUT` and `DELETE`.

When the `ApiPlatform\Core\Annotation\ApiResource` annotation is applied to an entity class, the following builtin CRUD
operations are automatically enabled:

*Collection operations*

Method | Mandatory | Description
-------|------------------------------------------
`GET`  | yes       | Retrieve the (paginated) list of elements
`POST` | no        | Create a new element

*Item operations*

Method   | Mandatory | Description
---------|------------------
`GET`    | yes       | Retrieve element
`PUT`    | no        | Update an element
`DELETE` | no        | Delete an element

## Enabling and disabling operations

If no operation is specified, all default CRUD operations are automatically registered. It is also possible - and recommended
for large projects - to define explicitly operations.

Keep in mind that `collectionOperations` and `itemOperations` behave independently. For instance, if you don't explicitly
configure operations for `collectionOperations`, `GET` and `POST` operations will be automatically registered, even if you
explicitly configure `itemOperations`. The reverse is also true.

Operations can be configured using annotations, XML or YAML. In the following examples, we enable only the builtin operation
for the `GET` method for both `collectionOperations` and `itemOperations` to create a readonly endpoint.

`itemOperations` and `collectionOperations` are arrays containing a list of operation. Each operation is defined by a key
corresponding to the name of the operation that can be anything you want and an array of properties as value.

<configurations>

```php
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

API Platform Core is smart enough to automatically register the applicable Symfony route referencing a builtin CRUD action
just by specifying the enabled HTTP method.

## Configuring operations

The URL, the HTTP method and the Hydra context passed to documentation generators of operations can be easily configured.

In the following examples, the builtin `GET` and `PUT` item operations are registered but they use custom URLs instead of
the default one guessed by API Platform from the resource name. The Hydra context is also override for the `PUT` operation.

<configurations>

```php
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

## Creating custom operations and controllers

API Platform can leverage the Symfony routing system to register custom operation related to custom controllers. Such custom
controllers can be any valid [Symfony controller](http://symfony.com/doc/current/book/controller.html), including standard
Symfony controllers extending the [`Symfony\Bundle\FrameworkBundle\Controller\Controller`](http://api.symfony.com/3.1/Symfony/Bundle/FrameworkBundle/Controller/Controller.html)
helper class.

However, API recommends to use **action classes** instead of typical Symfony controllers:

Internally, API Platform implements the [Action-Domain-Responder](https://github.com/pmjones/adr) pattern (ADR), a web-specific
refinement of [MVC](https://en.wikipedia.org/wiki/Model%E2%80%93view%E2%80%93controller).

The standard edition of API Platform also comes with an enhanced action system for Symfony pre-installed: [DunglasActionBundle](https://github.com/dunglas/DunglasActionBundle).
*DunglasActionBundle* eases the implementation of the ADR pattern with Symfony and improves the developer experience.

It automatically registers action classes stored in `src/*Bundle/Action` and `src/*Bundle/Controller` directories as autowired
services.

Thanks to the [autowiring](http://symfony.com/doc/current/components/dependency_injection/autowiring.html) feature of the
Symfony dependency injection container, services required by an action can be type-hinted in its controller, it will be automatically
instantiated and injected, without requiring to declare it explicitly.


---- Not finished below, see https://github.com/dunglas/api-platform/tree/custom_controller ----

<configurations>

```php
// src/AppBundle/Entity/Book.php

use ApiPlatform\Core\Annotation\ApiResource;

/**
 * @ApiResource(itemOperations={
 *     "get"={"method"="GET"},
 *     "put"={"method"="PUT"},
 *     "custom_get"={
 *        "path"="/products/{id}/custom",
 *        "method"="GET",
 *        "controller"="my_custom_controller",
 *        "hydra_context"={"@type"="hydra:Operation", "hydra:title"="A custom operation", "returns"="xmls:string"}
 *     }
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
        put:
            method: 'PUT'
        custom_get:
            method: 'GET'
            path: '/products/{id}/custom'
            controller: 'my_custom_controller'
            hydra_context:  # you can customize hydra context per operation
              "@type": "hydra:Operation"
              "hydra:title": "A custom operation"
              "returns": "xmls:string"
```

```xml
<!-- src/Acme/BlogBundle/Resources/config/resources.xml -->
<?xml version="1.0" encoding="UTF-8" ?>
<resources>
    <resource class="AppBundle\Entity\Book">
        <itemOperations type="collection">
            <operation key="get" method="GET" />
            <operation key="put" method="PUT" />
            <operation key="custom_get" method="GET" path="/products/{id}/custom" controller="my_custom_controller">
                <attribute key="hydra_context" type="collection">
                <attribute key="@type">hydra:Operation</attribute>
                <attribute key="hydra:title">A custom Operation</attribute>
                <attribute key="returns">xmls:string</attribute>
            </operation>
        </itemOperations>
    </resource>
</resources>
```

</configurations>

```yaml
products.custom_get:
    path:  '/products/{id}/custom'
    methods:  ['GET', 'HEAD']
    defaults:
        _controller:     'AppBundle:Custom:custom'
        _resource_class: 'AppBundle\Entity\Dummy'
        _item_operation_name: 'custom_get'
```

However, in the following example items operations will still be automatically registered. To disable them, simply override the itemOperation configuration:


Previous chapter: [NelmioApiDocBundle integration](nelmio-api-doc.md)<br>
Next chapter: [Data providers](data-providers.md)

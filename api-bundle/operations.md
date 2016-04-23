# Operations

By default, the following operations are automatically enabled:

*Collection*

| Method | Description                               |
|--------|-------------------------------------------|
| `GET`  | Retrieve the (paginated) list of elements |
| `POST` | Create a new element                      |

*Item*

| Method   | Description                               |
|----------|-------------------------------------------|
| `GET`    | Retrieve element (mandatory operation)    |
| `PUT`    | Update an element                         |
| `DELETE` | Delete an element                         |

## Creating custom operations

ApiPlatform allows to register custom operations for collections and items.
Custom operations allow to customize routing information (like the URL and the HTTP method),
the controller to use and a context that will be passed to documentation generators.

An operation is represented by an array having the following properties:

- `method` - GET, POST, PUT, DELETE
- `controller` - a custom controller service, defaults Api Platform controllers according to the method and whether it's a collection or an item operation
- `path` - your route path
- `route_name` - if this is specified when using symfony, it'll not try to register the route and assume it exists and is registered

There are always two kind of operations, `collectionOperations` and `itemOperations`. `collectionOperations` will act on your entity collection, by default two routes are implemented (POST, GET).
`itemOperations` on the other hand are representing your collection items and defines 3 default routes (GET, PUT, DELETE).

If you want to use custom controller action, [refer to the dedicated documentation](controllers.md).

<configurations>

```php
<?php
use ApiPlatform\Core\Annotation\Resource;
use Doctrine\ORM\Mapping as ORM;

/**
 * @Resource(itemOperations={
 *     "get"={"method"="GET"},
 *     "put"={"method"="PUT"},
 *     "custom_get"={
 *        "path"="/products/{id}/custom",
 *        "method"="GET",
 *        "controller"="my_custom_controller",
 *        "hydra_context"={"@type"="hydra:Operation", "hydra:title"="A custom operation", "returns"="xmls:string"}
 *     }
 * })
 * @ORM\Entity
 */
class Product
{
//...
```

```yaml
# src/AppBundle/Resources/config/resources.yml
resources:
  product:
    class: 'AppBundle\Entity\Product'
    itemOperations:
        get:
            method: 'GET' # nothing more to add if we want to keep the default controller
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
# src/Acme/BlogBundle/Resources/config/resources.xml
<?xml version="1.0" encoding="UTF-8" ?>
<resources>
        <resource class="AppBundle\Entity\Product">
            <itemOperations type="collection">
                    <operation key="get" method="GET" />
                    <operation key="put" method="PUT" />
                    <operation key="custom_get" method="GET" path="/products/{id}/custom" controller="my_custom_controller">
                      <attribute key="hydra_context" type="collection">
                        <attribute key="@type">hydra:Operation</attribute>
                        <attribute key="hydra:title">A custom Operation</attribute>
                        <attribute key="returns">xmls:string</attribute>
                      </attribute>
                    </operation>
            </itemOperations>
        </resource>
</resources>
```

</configurations>

Additionally to the default generated `GET` and `PUT` operations, this definition will expose a new `GET` operation for
the `/products/{id}/custom` URL. When this URL is opened, the `my_custom_controller` controller is called.

## Disabling operations

If you want to disable operations, you need to declare operations in your configuration.

The following `Resource` definition exposes a `GET` operation for it's collection but not the `POST` one:

<configurations>
```yaml
# src/AppBundle/Resources/config/resources.yml
resources:
  product:
    class: 'AppBundle\Entity\Product'
    collectionOperations:
       get:
        method: 'GET'
```

```xml
# src/Acme/BlogBundle/Resources/config/resources.xml
<?xml version="1.0" encoding="UTF-8" ?>
<resources>
        <resource class="AppBundle\Entity\Product">
          <collectionOperations type="collection">
            <operation key="get" method="GET" />
          </collectionOperations>
        </resource>
</resources>
```

```php
<?php
use ApiPlatform\Core\Annotation\Resource;
use Doctrine\ORM\Mapping as ORM;

/**
 * @Resource(collectionOperations={"get"={"method"="GET"}})
 * @ORM\Entity
 */
class Product
{
//...
```
</configurations>

However, in the following example items operations will still be automatically registered. To disable them, simply override the itemOperation configuration:

<configurations>
```yaml
# src/AppBundle/Resources/config/resources.yml
resources:
  product:
    class: 'AppBundle\Entity\Product'
    collectionOperations:
       get:
        method: 'GET'
    itemOperations: []
```

```xml
# src/Acme/BlogBundle/Resources/config/resources.xml
<?xml version="1.0" encoding="UTF-8" ?>
<resources>
        <resource class="AppBundle\Entity\Product">
          <collectionOperations type="collection">
            <operation key="get" method="GET" />
          </collectionOperations>
          <itemOperations type="collection" />
        </resource>
</resources>
```

```php
<?php
use ApiPlatform\Core\Annotation\Resource;
use Doctrine\ORM\Mapping as ORM;

/**
 * @Resource(collectionOperations={"get"={"method"="GET"}}, itemOperations={})
 * @ORM\Entity
 */
class Product
{
//...
```
</configurations>


Previous chapter: [NelmioApiDocBundle integration](nelmio-api-doc.md)<br>
Next chapter: [Data providers](data-providers.md)

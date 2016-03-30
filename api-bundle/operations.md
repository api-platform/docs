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


## Disabling operations

If you want to disable some operations (e.g. the `DELETE` operation), you must use the @Resource annotation.

The following `Resource` definition exposes a `GET` operation for it's collection but not the `POST` one:

```php
/**
 * Product Class.
 *
 * @author Kévin Dunglas <dunglas@gmail.com>
 *
 * @Resource(collectionOperations={
 *     "get"={"method"="GET"},
 * })
 * @ORM\Entity
 */
class Product
```

However, in the following example items operations will still be automatically registered. To disable them, call @Register
with an empty itemOperations as first parameter:

```php
/**
 * Product Class.
 *
 * @author Kévin Dunglas <dunglas@gmail.com>
 *
 * @Resource(itemOperations={})
 * @ORM\Entity
 */
class Product
```

## Creating custom operations

ApiPlatformBundle allows to register custom operations for collections and items.
Custom operations allow to customize routing information (like the URL and the HTTP method),
the controller to use (default to the built-in action of the `Resource` applicable
for the given HTTP method) and a context that will be passed to documentation generators.


You will have to create a custom route with a custom controller action using the `@Resource`.

If you want to use custom controller action, [refer to the dedicated documentation](controllers.md).

```yaml
products.custom_get:
    path:  '/products/{id}/custom'
    methods:  ['GET', 'HEAD']
    defaults:
        _controller:     'AppBundle:Custom:custom'
        _resource_class: 'AppBundle\Entity\Dummy'
        _operation_name: 'custom_get'

```

Additionally to the default generated `GET` and `PUT` operations, this definition will expose a new `GET` operation for
the `/products/{id}/custom` URL. When this URL is opened, the `AppBundle:Custom:custom` controller is called.

Previous chapter: [NelmioApiDocBundle integration](nelmio-api-doc.md)<br>
Next chapter: [Data providers](data-providers.md)

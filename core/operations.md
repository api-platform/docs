# Operations

API Platform relies on the concept of operations. Operations can be applied to a resource exposed by the API. From
an implementation point of view, an operation is a link between a resource, a route and its related controller.

<p align="center" class="symfonycasts"><a href="https://symfonycasts.com/screencast/api-platform/operations?cid=apip"><img src="../symfony/images/symfonycasts-player.png" alt="Operations screencast"><br>Watch the Operations screencast</a></p>

API Platform automatically registers typical [CRUD](https://en.wikipedia.org/wiki/Create,_read,_update_and_delete) operations
and describes them in the exposed documentation (Hydra and Swagger). It also creates and registers routes
for these operations in the Symfony routing system, if available, or in the Laravel routing system,
should that be the case.

The behavior of built-in operations is briefly presented in the [Getting started](getting-started.md#mapping-the-entities)
guide.

The list of enabled operations can be configured on a per-resource basis. Creating custom operations on specific routes
is also possible.

There are two types of operations: collection operations and item operations.

Collection operations act on a collection of resources. By default two operations are implemented: `POST` and `GET`. Item
operations act on an individual resource. Three default operation are defined: `GET`, `DELETE` and `PATCH`. `PATCH` is supported
with [JSON Merge Patch (RFC 7396)](https://www.rfc-editor.org/rfc/rfc7386), or [using the JSON:API format](https://jsonapi.org/format/#crud-updating), as required by the specification.

The `PUT` operation is also supported, but is not registered by default.

When the `ApiPlatform\Metadata\ApiResource` annotation is applied to an entity class, the following built-in CRUD
operations are automatically enabled:

Collection operations:

| Method | Mandatory | Description                               | Registered by default |
| ------ | --------- | ----------------------------------------- | --------------------- |
| `GET`  | yes       | Retrieve the (paginated) list of elements | yes                   |
| `POST` | no        | Create a new element                      | yes                   |

Item operations:

| Method   | Mandatory | Description                                | Registered by default |
| -------- | --------- | ------------------------------------------ | --------------------- |
| `GET`    | yes       | Retrieve an element                        | yes                   |
| `PUT`    | no        | Replace an element                         | no                    |
| `PATCH`  | no        | Apply a partial modification to an element | yes                   |
| `DELETE` | no        | Delete an element                          | yes                   |

> [!NOTE]
> The `PATCH` method must be enabled explicitly in the configuration, refer to the [Content Negotiation](content-negotiation.md) section for more information.

---

> [!NOTE]
> With JSON Merge Patch, the [null values will be skipped](https://symfony.com/doc/current/components/serializer.html#skipping-null-values) in the response.

---

> [!NOTE]
> Current `PUT` implementation behaves more or less like the `PATCH` method.
> Existing properties not included in the payload are **not** removed, their current values are preserved.
> To remove an existing property, its value must be explicitly set to `null`.

## Enabling and Disabling Operations

If no operation is specified, all default CRUD operations are automatically registered. It is also possible - and recommended
for large projects - to define operations explicitly.

Keep in mind that once you explicitly set up an operation, the automatically registered CRUD will no longer be.
If you declare even one operation manually, such as `#[GET]`, you must declare the others manually as well if you need them.

Operations can be configured using attributes, XML or YAML. In the following examples, we enable only the built-in operation
for the `GET` method for both `collection` and `item` to create a readonly endpoint.

If the operation's name matches a supported HTTP method (`GET`, `POST`, `PUT`, `PATCH` or `DELETE`), the corresponding `method` property
will be automatically added.

> [!TIP]
> The `#[GetCollection]` attribute is an alias for `#[Get(collection: true)]`

---

> [!NOTE]
> In Symfony we use the term “entities”, while the following documentation is mostly for Laravel “models”.

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
resources:
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
resources:
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
resources:
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
resources:
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
resources:
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

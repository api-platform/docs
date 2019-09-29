# Security

The API Platform security layer is built on top of the [Symfony Security component](http://symfony.com/doc/current/book/security.html).
All its features, including [global access control directives](http://symfony.com/doc/current/book/security.html#securing-url-patterns-access-control) are supported.
API Platform also provides convenient [access control expressions](https://symfony.com/doc/current/expressions.html#security-complex-access-controls-with-expressions) which you can apply at resource and operation level:

```php
<?php
// api/src/Entity/Book.php

use ApiPlatform\Core\Annotation\ApiResource;
use Doctrine\ORM\Mapping as ORM;
use Symfony\Component\Validator\Constraints as Assert;

/**
 * Secured resource.
 *
 * @ApiResource(
 *     attributes={"security"="is_granted('ROLE_USER')"},
 *     collectionOperations={
 *         "get",
 *         "post"={"security"="is_granted('ROLE_ADMIN')"}
 *     },
 *     itemOperations={
 *         "get",
 *         "put"={"security"="is_granted('ROLE_ADMIN') or object.owner == user"},
 *     }
 * )
 * @ORM\Entity
 */
class Book
{
    /**
     * @var int
     *
     * @ORM\Column(type="integer")
     * @ORM\Id
     * @ORM\GeneratedValue(strategy="AUTO")
     */
    private $id;

    /**
     * @var string The title
     *
     * @ORM\Column
     * @Assert\NotBlank
     */
    public $title;

    /**
     * @var User The owner
     *
     * @ORM\ManyToOne(targetEntity=User::class)
     */
    public $owner;

    // ...
}
```

Alternatively, using YAML:

```yaml
# api/config/api_platform/resources.yaml
App\Entity\Book:
    attributes:
        security: 'is_granted("ROLE_USER")'
    collectionOperations:
        get: ~
        post:
            security: 'is_granted("ROLE_ADMIN")'
    itemOperations:
        get: ~
        put:
            security_: 'is_granted("ROLE_ADMIN") or object.owner == user'
```

In this example:

* The user must be logged in to interact with `Book` resources (configured at the resource level)
* Only users having [the role](https://symfony.com/doc/current/security.html#roles) `ROLE_ADMIN` can create a new resource (configured on the `post` operation)
* Only users having the `ROLE_ADMIN` or owning the current object can replace an existing book (configured on the `put` operation)

Available variables are `user` (the current logged in object, if any), and `object` (the current resource, or collection of ressources for collection operations).

Access control checks in the `security` attribute are always executed before the [denormalization step](serialization.md).
It means than for `PUT` requests, `object` doesn't contain the value submitted by the user, but values currently stored in [the persistence layer](data-persisters.md).

## Executing Access Control Rules After Denormalization

In some cases, it might be useful to execute a security after the denormalization step.
To do so, use the `security_post_denormalize` attribute:

```php
<?php
// src/Entity/Book.php

namespace App\Entity;

use ApiPlatform\Core\Annotation\ApiResource;

/**
 * @ApiResource(
 *     itemOperations={
 *         "get",
 *         "put"={"security_post_denormalize"="is_granted('ROLE_ADMIN') or (object.owner == user and previous_object.owner == user)"},
 *     }
 * )
 */
class Book
{
    // ...
}
```

Alternatively, using YAML:

```yaml
# api/config/api_platform/resources.yaml
App\Entity\Book:
    itemOperations:
        get: ~
        put:
            security_post_denormalize: "is_granted('ROLE_ADMIN') or (object.owner == user and previous_object.owner == user)"
    # ...
```

This time, the `object` variable contains data that have been extracted from the HTTP request body during the denormalization process.
However, the object is not persisted yet.

Additionally, in some cases you need to perform security checks on the original data. For example here, only the actual owner should be allowed to edit their book. In these cases, you can use the `previous_object` variable which contains the object that was read from the data provider.

The value in the `previous_object` variable is cloned from the original object.
Note that, by default, this clone is not a deep one (it doesn't clone relationships, relationships are references).
To make a deep clone, [implement `__clone` method](https://www.php.net/manual/en/language.oop5.cloning.php) in the concerned resource class.

## Hooking Custom Permission Checks Using Voters

The easiest and recommended way to hook custom access control logic is [to write Symfony Voter classes](https://symfony.com/doc/current/security/voters.html). Your custom voters will automatically be used in security expressions through the `is_granted()` function.

## Configuring the Access Control Error Message

By default when API requests are denied, you will get the "Access Denied" message.
You can change it by configuring the `security_message` attribute.

For example:

```php
<?php
// api/src/Entity/Book.php

namespace App\Entity;

use ApiPlatform\Core\Annotation\ApiResource;

/**
 * ...
 * @ApiResource(
 *     attributes={"security"="is_granted('ROLE_USER')"},
 *     collectionOperations={
 *         "post"={"security"="is_granted('ROLE_ADMIN')", "security_message"="Only admins can add books."}
 *     },
 *     itemOperations={
 *         "get"={"security"="is_granted('ROLE_USER') and object.owner == user", "security_message"="Sorry, but you are not the book owner."}
 *     }
 * )
 */
class Book
{
    // ...
}
```

Alternatively, using YAML:

```yaml
# api/config/api_platform/resources.yaml
App\Entity\Book:
    attributes:
        security: 'is_granted("ROLE_USER")'
    collectionOperations:
        post:
            method: 'POST'
            security: 'is_granted("ROLE_ADMIN")'
            security_message: 'Only admins can add books.'
    itemOperations:
        get:
            method: 'GET'
            security: 'is_granted("ROLE_USER") and object.owner == user'
            security_message: 'Sorry, but you are not the book owner.'
    # ...
```

## Filtering Collection According to the Current User Permissions

Filtering collections according to the role or permissions of the current user must be done directly at [the data provider](data-providers.md) level. For instance, when using the built-in adapters for Doctrine ORM, MongoDB and ElasticSearch, removing entries from a collection should be done using [extensions](extensions.md).
Extensions allow to customize the generated DQL/Mongo/Elastic/... query used to retrieve the collection (e.g. add `WHERE` clauses depending of the currently connected user) instead of using access control expressions. As extensions are services, you can [inject the Symfony `Security` class](https://symfony.com/doc/current/security.html#b-fetching-the-user-from-a-service) into them to access to current user's roles and permissions.

If you use [custom data providers](data-providers.md), you'll have to implement the filtering logic according to the persistence layer you rely on.

## Disabling Operations

To completely disable some operations from your application, refer to the [disabling operations](operations.md#enabling-and-disabling-operations)
section.

## Changing Serialization Groups Depending of the Current User

See [how to dynamically change](serialization.md#changing-the-serialization-context-dynamically) the current Serializer context according to the current logged in user.

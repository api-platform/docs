# Security

If you have yet to set up authentication for your API, refer to the section on [JWT authentication](jwt.md).

To completely disable some operations from your application, refer to the [disabling operations](operations.md#enabling-and-disabling-operations)
section.

Using API Platform, you can leverage all security features provided by the [Symfony Security component](http://symfony.com/doc/current/book/security.html).
For instance, if you wish to restrict the access of some endpoints, you can use [access control directives](http://symfony.com/doc/current/book/security.html#securing-url-patterns-access-control).

Since 2.1, you can add security through [Symfony's access control expressions](https://symfony.com/doc/current/expressions.html#security-complex-access-controls-with-expressions) in your entities.

Here is an example:

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
 *         "get"={"security"="is_granted('ROLE_USER') and object.owner == user"},
 *         "put"={"access_control"="is_granted('ROLE_USER') and previous_object.owner == user"},
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

This example is only going to allow fetching the book related to the current user. If the user tries to fetch a book which is not
linked to his account, it will not return the resource. In addition, only admins are able to create books which means
that a user could not create a book.

Additionally, in some cases you need to perform security checks on the original data. For example here, only the actual owner should be allowed to edit their book. In these cases, you can use the `previous_object` variable which contains the object that was read from the data provider.

N.B `previous_object` is cloned from the original object. Note that this clone is not a deep one (it doesn't clone relationships, relationships are references), to [make a deep clone](https://www.php.net/manual/fr/language.oop5.cloning.php#object.clone) implement `__clone` method in the concerned resource class.

It is also possible to use the [event system](events.md) for more advanced logic or even [custom actions](operations.md#creating-custom-operations-and-controllers)
if you really need to.

## Configuring the Access Control Message

By default when API requests are denied, you will get the "Access Denied" message.
You can change it by configuring the "security\_message" attribute.

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

## Execute security after denormalization

The "security" attribute is executed before the object denormalization. For some cases, it might be useful to execute
a security after the denormalization.
To do so, prefer using "security\_post\_denormalize", which allows you to use the "previous\_object" as the original object:

```php
<?php
// src/Entity/Book.php

namespace App\Entity;

use ApiPlatform\Core\Annotation\ApiResource;

/**
 * ...
 * @ApiResource(
 *     attributes={"security"="is_granted('ROLE_USER')"},
 *     itemOperations={
 *         "get",
 *         "put"={"security_post_denormalize"="is_granted("ROLE_USER") and previous_object.owner == user"}
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
    itemOperations:
        get: ~
        put:
            method: 'PUT'
            security_post_denormalize: 'is_granted("ROLE_USER") and previous_object.owner == user'
    # ...
```

In access control expressions for collections, the `object` variable contains the list of resources that will be serialized.
To remove entries from a collection, you should implement [a Doctrine extension](extensions.md) to customize the generated DQL query (e.g. add `WHERE` clauses depending of the currently connected user) instead of using access control expressions.

If you use [custom data providers](data-providers.md), you'll have to implement the filtering logic according to the persistence layer you rely on.

# Security

To completely disable some operations from your application, refer to the [disabling operations](operations.md#enabling-and-disabling-operations)
section.

Using API Platform, you can leverage all security features provided by the [Symfony Security component](http://symfony.com/doc/current/book/security.html).
For instance, if you wish to restrict the access of some endpoints, you can use [access controls directives](http://symfony.com/doc/current/book/security.html#securing-url-patterns-access-control).

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
 *     attributes={"access_control"="is_granted('ROLE_USER')"},
 *     collectionOperations={
 *         "get",
 *         "post"={"access_control"="is_granted('ROLE_ADMIN')"}
 *     },
 *     itemOperations={
 *         "get"={"access_control"="is_granted('ROLE_USER') and object.owner == user"}
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
    public $id;

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
}
```

This example is going to allow only fetching the book related to the current user. If he tries to fetch a book which is not
linked to his account, that will not return the resource. In addition, only admins are able to create books which means
that a user could not create a book.

It is also possible to use the [event system](events.md) for more advanced logic or even [custom actions](operations.md#creating-custom-operations-and-controllers)
if you really need to.

## Configuring the Access Control Message

By default when API request will be denied you will get the "Access Denied." message.
You can change it by configuring "access\_control\_message" attribute.

For example:

```php
<?php
// src/Entity/Book.php

namespace App\Entity;

use ApiPlatform\Core\Annotation\ApiResource;

/**
 * ...
 * @ApiResource(
 *     attributes={"access_control"="is_granted('ROLE_USER')"},
 *     collectionOperations={
 *         "post"={"access_control"="is_granted('ROLE_ADMIN')", "access_control_message"="Only admins can add books."}
 *     },
 *     itemOperations={
 *         "get"={"access_control"="is_granted('ROLE_USER') and object.owner == user", "access_control_message"="Sorry, but you are not the book owner."}
 *     }
 * )
 */
class Book
{
    // ...
}
```

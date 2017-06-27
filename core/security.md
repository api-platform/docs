# Security

To completely disable some operations from your application, refer to the [disabling operations](operations.md#enabling-and-disabling-operations)
section.

Using API Platform, you can leverage all security features provided by the [Symfony Security component](http://symfony.com/doc/current/book/security.html).
For instance, if you wish to restrict the access of some endpoints, you can use [access controls directives](http://symfony.com/doc/current/book/security.html#securing-url-patterns-access-control).

Since 2.1, you can add security through [Symfony's access control expressions](https://symfony.com/doc/current/expressions.html#security-complex-access-controls-with-expressions) in your entities.

Here is an example:

```php
use ApiPlatform\Core\Annotation\ApiResource;
use Doctrine\ORM\Mapping as ORM;
use Symfony\Component\Validator\Constraints as Assert;

/**
 * Secured resource.
 *
 * @ApiResource(
 *     attributes={"is_granted"="has_role('ROLE_USER')"},
 *     collectionOperations={
 *         "get"={"method"="GET"},
 *         "post"={"method"="POST", "is_granted"="has_role('ROLE_ADMIN')"}
 *     },
 *     itemOperations={
 *         "get"={"method"="GET", "is_granted"="has_role('ROLE_USER') and object.getOwner() == user"}
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
}
```

It is also possible to use the [event system](events.md) for more advanced logic or even [custom actions](operations.md#creating-custom-operations-and-controllers)
if you really need to.

Previous chapter: [Extensions](extensions.md)

Next chapter: [Performance](performance.md)

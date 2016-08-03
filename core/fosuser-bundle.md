# FOSUser Bundle Integration

API Platform Core is shipped with a bridge for [FOSUserBundle](https://github.com/FriendsOfSymfony/FOSUserBundle). If the
FOSUser bundle is enabled, this bridge will use its `UserManager` to create, update and delete user resources.

## Creating a `User` Entity with Serialization Groups

Here's an example of declaration of a [Doctrine ORM User class](https://github.com/FriendsOfSymfony/FOSUserBundle/blob/master/Resources/doc/index.rst#a-doctrine-orm-user-class).
You need to use serialization groups to hide some properties like `plainPassword` (only in read) and `password`. The properties
shown are handled with the [`normalization_context`](serialization-groups-and-relations.md#normalization), while the properties
you can modify are handled with [`denormalization_context`](serialization-groups-and-relations.md#denormalization).

First register the following service:

```php
// src/AppBundle/Entity/User.php

namespace AppBundle\Entity;

use ApiPlatform\Core\Annotation\ApiResource;
use Doctrine\ORM\Mapping as ORM;
use FOS\UserBundle\Model\User as BaseUser;
use FOS\UserBundle\Model\UserInterface;
use Symfony\Component\Serializer\Annotation\Groups;

/**
 * @ORM\Entity
 * @ApiResource(attributes={
 *     "normalization_context"={"groups"={"user", "user-read"}},
 *     "denormalization_context"={"groups"={"user", "user-write"}}
 * })
 */
class User extends BaseUser
{
    /**
     * @ORM\Id
     * @ORM\Column(type="integer")
     * @ORM\GeneratedValue(strategy="AUTO")
     */
    protected $id;

    /**
     * @Groups({"user"})
     */
    protected $email;

    /**
     * @ORM\Column(type="string", length=255, nullable=true)
     * @Groups({"user"})
     */
    protected $fullname;

    /**
     * @Groups({"user-write"})
     */
    protected $plainPassword;

    /**
     * @Groups({"user"})
     */
    protected $username;

    public function setFullname($fullname)
    {
        $this->fullname = $fullname;

        return $this;
    }
    public function getFullname()
    {
        return $this->fullname;
    }

    public function isUser(UserInterface $user = null)
    {
        return $user instanceof self && $user->id === $this->id;
    }
}
```

Previous chapter: [Path Naming Strategy](path-naming-strategy.md)<br>
Next chapter: [AngularJS integration](angularjs-integration.md)

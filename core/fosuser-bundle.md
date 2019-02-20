# FOSUserBundle Integration

API Platform Core is shipped with a bridge for [FOSUserBundle](https://github.com/FriendsOfSymfony/FOSUserBundle).
If the FOSUser bundle is enabled, this bridge will use its `UserManager` to create, update and delete user resources.

Note: FOSUserBundle is not well suited for APIs. We strongly encourage you to use the [Doctrine user provider](https://symfony.com/doc/current/security/entity_provider.html)
shipped with Symfony or to [create a custom user provider](http://symfony.com/doc/current/security/custom_provider.html)
instead of using this bundle.

## Installing the Bundle

The installation procedure of the FOSUserBundle is described [in the main Symfony docs](https://symfony.com/doc/master/bundles/FOSUserBundle/index.html)

You can:

* Skip the [step 3 (Create your User class)](https://symfony.com/doc/master/bundles/FOSUserBundle/index.html#step-3-create-your-user-class)
and use the class provided in the next paragraph to set up serialization groups the correct way
* Skip the [step 4 (Configure your application's security.yml)](https://symfony.com/doc/master/bundles/FOSUserBundle/index.html#step-4-configure-your-application-s-security-yml)
if you are planning to [use a JWT-based authentication using `LexikJWTAuthenticationBundle`](jwt.md)

If you are using the API Platform Standard Edition, you will need to enable the form services in the symfony framework
configuration options:

```yaml
# api/config/packages/framework.yaml
framework:
    form: { enabled: true }
```

## Enabling the Bridge

To enable the provided bridge with FOSUserBundle, you need to add the following configuration to api-platform:
```yaml
# api/config/packages/api_platform.yaml
api_platform:
    enable_fos_user: true
```

## Creating a `User` Entity with Serialization Groups

Here's an example of declaration of a [Doctrine ORM User class](https://github.com/FriendsOfSymfony/FOSUserBundle/blob/master/Resources/doc/index.rst#a-doctrine-orm-user-class).
There's also an example for a [Doctrine MongoDB ODM](https://github.com/FriendsOfSymfony/FOSUserBundle/blob/master/Resources/doc/index.rst#b-mongodb-user-class).
You need to use serialization groups to hide some properties like `plainPassword` (only in read) and `password`. The properties
shown are handled with the [`normalization_context`](serialization.md#normalization), while the properties
you can modify are handled with [`denormalization_context`](serialization.md#denormalization).

Create your User entity with serialization groups:

```php
<?php
// api/src/Entity/User.php

namespace App\Entity;

use ApiPlatform\Core\Annotation\ApiResource;
use Doctrine\ORM\Mapping as ORM;
use FOS\UserBundle\Model\User as BaseUser;
use FOS\UserBundle\Model\UserInterface;
use Symfony\Component\Serializer\Annotation\Groups;

/**
 * @ORM\Entity
 * @ORM\Table(name="fos_user")
 * @ApiResource(
 *     normalizationContext={"groups"={"user", "user:read"}},
 *     denormalizationContext={"groups"={"user", "user:write"}}
 * )
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
     * @Groups({"user:write"})
     */
    protected $plainPassword;

    /**
     * @Groups({"user"})
     */
    protected $username;

    public function setFullname(?string $fullname): void
    {
        $this->fullname = $fullname;
    }

    public function getFullname(): ?string
    {
        return $this->fullname;
    }

    public function isUser(?UserInterface $user = null): bool
    {
        return $user instanceof self && $user->id === $this->id;
    }
}
```

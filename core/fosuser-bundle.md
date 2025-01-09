# FOSUserBundle Integration

## ⚠️ Deprecated: this integration is deprecated and will be removed in API Platform 3

FOSUserBundle is not well suited for APIs. We strongly encourage you to use the [Doctrine user provider](https://symfony.com/doc/current/security/user_provider.html#entity-user-provider)
shipped with Symfony or to [create a custom user provider](https://symfony.com/doc/current/security/user_provider.html#creating-a-custom-user-provider)
instead of using this bundle.

API Platform Core is shipped with a bridge for [FOSUserBundle](https://github.com/FriendsOfSymfony/FOSUserBundle).
If the FOSUser bundle is enabled, this bridge will use its `UserManager` to create, update and delete user resources.

<p align="center" class="symfonycasts"><a href="https://symfonycasts.com/screencast/api-platform/user-entity?cid=apip"><img src="../distribution/images/symfonycasts-player.png" alt="User Entity screencast"><br>Watch the User Entity screencast</a></p>

## Installing the Bundle

The installation procedure of the FOSUserBundle is described [in the main Symfony docs](https://symfony.com/doc/master/bundles/FOSUserBundle/index.html)

You can:

* Skip [step 3 (Create your User class)](https://symfony.com/doc/master/bundles/FOSUserBundle/index.html#step-3-create-your-user-class)
and use the class provided in the next paragraph to set up serialization groups the correct way
* Skip [step 4 (Configure your application's security.yml)](https://symfony.com/doc/master/bundles/FOSUserBundle/index.html#step-4-configure-your-application-s-security-yml)
if you are planning to [use a JWT-based authentication using `LexikJWTAuthenticationBundle`](jwt.md)

If you are using the API Platform Standard Edition, you will need to enable the form services in the symfony framework
configuration options:

```yaml
# config/packages/framework.yaml
framework:
    form: { enabled: true }
```

## Enabling the Bridge

To enable the provided bridge with FOSUserBundle, you need to add the following configuration to API Platform:

```yaml
# config/packages/api_platform.yaml
api_platform:
    enable_fos_user: true
```

## Creating a `User` Entity with Serialization Groups

Here's an example of declaration of a [Doctrine ORM User class](https://github.com/FriendsOfSymfony/FOSUserBundle/blob/master/Resources/doc/index.rst#a-doctrine-orm-user-class).
There's also an example for a [Doctrine MongoDB ODM](https://github.com/FriendsOfSymfony/FOSUserBundle/blob/master/Resources/doc/index.rst#b-mongodb-user-class).
You need to use serialization groups to hide some properties like `plainPassword` (only in read) and `password`. The properties
shown are handled with [`normalization_context`](serialization.md#normalization), while the properties
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

#[ORM\Entity]
#[ORM\Table(name: 'fos_user')]
#[ApiResource(
    normalizationContext: ["groups" => ["user", "user:read"]],
    denormalizationContext: ["groups" => ["user", "user:write"]]
)]
class User extends BaseUser
{
    #[ORM\Id, ORM\Column, ORM\GeneratedValue]
    protected ?int $id = null;

    #[Groups("user")]
    protected string $email;

    #[ORM\Column(nullable: true)] 
    #[Groups("user")]
    protected string $fullname;

    #[Groups("user:write")]
    protected string $plainPassword;

    #[Groups("user")]
    protected string $username;

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

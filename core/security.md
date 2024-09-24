# Security

The API Platform security layer is built on top of the [Symfony Security component](https://symfony.com/doc/current/security.html).
All its features, including [global access control directives](https://symfony.com/doc/current/security.html#securing-url-patterns-access-control) are supported.
API Platform also provides convenient [access control expressions](https://symfony.com/doc/current/expressions.html#security-complex-access-controls-with-expressions) which you can apply at resource and operation level.

<p align="center" class="symfonycasts"><a href="https://symfonycasts.com/screencast/api-platform-security/?cid=apip"><img src="../distribution/images/symfonycasts-player.png" alt="Security screencast"><br>Watch the Security screencast</a></p>

<code-selector>

```php
<?php
// api/src/Entity/Book.php
namespace App\Entity;

use ApiPlatform\Metadata\ApiResource;
use ApiPlatform\Metadata\Get;
use ApiPlatform\Metadata\GetCollection;
use ApiPlatform\Metadata\Post;
use ApiPlatform\Metadata\Put;
use Doctrine\ORM\Mapping as ORM;
use Symfony\Component\Validator\Constraints as Assert;

/**
 * Secured resource.
 */
#[ApiResource(security: "is_granted('ROLE_USER')")]
#[Get]
#[Put(security: "is_granted('ROLE_ADMIN') or object.owner == user")]
#[GetCollection]
#[Post(security: "is_granted('ROLE_ADMIN')")]
#[ORM\Entity]
class Book
{
    #[ORM\Id, ORM\Column, ORM\GeneratedValue]
    private ?int $id = null;

    #[ORM\Column]
    #[Assert\NotBlank]
    public string $title;

    #[ORM\ManyToOne]
    public User $owner;

    // ...
}
```

```yaml
# api/config/api_platform/resources.yaml
resources:
    App\Entity\Book:
        security: 'is_granted("ROLE_USER")'
        operations:
            ApiPlatform\Metadata\GetCollection: ~
            ApiPlatform\Metadata\Post:
                security: 'is_granted("ROLE_ADMIN")'
            ApiPlatform\Metadata\Get: ~
            ApiPlatform\Metadata\Put:
                security: 'is_granted("ROLE_ADMIN") or object.owner == user'
```

</code-selector>

Resource signature can be modified at the property level as well:

<code-selector>

```php
<?php
// api/src/Entity/Book.php
namespace App\Entity;

use ApiPlatform\Metadata\ApiProperty;

class Book
{
    //...

    /**
     * @var string Property viewable and writable only by users with ROLE_ADMIN
     */
    #[ApiProperty(security: "is_granted('ROLE_ADMIN')", securityPostDenormalize: "is_granted('UPDATE', object)")]
    private $adminOnlyProperty;
}
```

```yaml
# api/config/api_platform/resources/Book.yaml
properties:
    App\Entity\Book:
        adminOnlyProperty:
            security: 'is_granted("ROLE_ADMIN")'
```

</code-selector>

In this example:

* The user must be logged in to interact with `Book` resources (configured at the resource level)
* Only users having [the role](https://symfony.com/doc/current/security.html#roles) `ROLE_ADMIN` can create a new resource (configured on the `post` operation)
* Only users having the `ROLE_ADMIN` or owning the current object can replace an existing book (configured on the `put` operation)
* Only users having the `ROLE_ADMIN` can view or modify the `adminOnlyProperty` property. Only users having the `ROLE_ADMIN` can create a new resource specifying `adminOnlyProperty` value.
* Only users that are granted the `UPDATE` attribute on the book (via a voter) can write to the field

Available variables are:

* `user`: the current logged in object, if any
* `object`: the current resource class during denormalization, the current resource during normalization, or collection of resources for collection operations
* `previous_object`: (`securityPostDenormalize` only) a clone of `object`, before modifications were made - this is `null` for create operations
* `request` (only at the resource level): the current request

Access control checks in the `security` attribute are always executed before the [denormalization step](serialization.md).
It means that for `PUT` or `PATCH` requests, `object` doesn't contain the value submitted by the user, but values currently stored in [the persistence layer](state-processors.md).

## Executing Access Control Rules After Denormalization

In some cases, it might be useful to execute a security after the denormalization step.
To do so, use the `securityPostDenormalize` attribute:

<code-selector>

```php
<?php
// api/src/Entity/Book.php
namespace App\Entity;

use ApiPlatform\Metadata\ApiResource;
use ApiPlatform\Metadata\Get;
use ApiPlatform\Metadata\Put;

#[ApiResource]
#[Get]
#[Put(securityPostDenormalize: "is_granted('ROLE_ADMIN') or (object.owner == user and previous_object.owner == user)")]
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
            ApiPlatform\Metadata\Get: ~
            ApiPlatform\Metadata\GetCollectionPut:
                securityPostDenormalize: "is_granted('ROLE_ADMIN') or (object.owner == user and previous_object.owner == user)"
        # ...
```

</code-selector>

This time, the `object` variable contains data that have been extracted from the HTTP request body during the denormalization process.
However, the object is not persisted yet.

Additionally, in some cases you need to perform security checks on the original data. For example here, only the actual owner should be allowed to edit their book. In these cases, you can use the `previous_object` variable which contains the object that was read from the state provider.

The value in the `previous_object` variable is cloned from the original object.
Note that, by default, this clone is not a deep one (it doesn't clone relationships, relationships are references).
To make a deep clone, [implement `__clone` method](https://www.php.net/manual/en/language.oop5.cloning.php) in the concerned resource class.

## Hooking Custom Permission Checks Using Voters

The easiest and recommended way to hook custom access control logic is [to write Symfony Voter classes](https://symfony.com/doc/current/security/voters.html). Your custom voters will automatically be used in security expressions through the `is_granted()` function.

In order to give the current `object` to your voter, use the expression `is_granted('READ', object)`

For example:

<code-selector>

```php
<?php
// api/src/Entity/Book.php
namespace App\Entity;

use ApiPlatform\Metadata\ApiResource;
use ApiPlatform\Metadata\Delete;
use ApiPlatform\Metadata\Get;
use ApiPlatform\Metadata\GetCollection;
use ApiPlatform\Metadata\Post;
use ApiPlatform\Metadata\Put;

#[ApiResource(security: "is_granted('ROLE_USER')")]
#[Get(security: "is_granted('BOOK_READ', object)")]
#[Put(security: "is_granted('BOOK_EDIT', object)")]
#[Delete(security: "is_granted('BOOK_DELETE', object)")]
#[GetCollection]
#[Post(securityPostDenormalize: "is_granted('BOOK_CREATE', object)")]
class Book
{
    // ...
}
```

```yaml
# api/config/api_platform/resources/Book.yaml
App\Entity\Book:
    security: 'is_granted("ROLE_USER")'
    operations:
        ApiPlatform\Metadata\GetCollection: ~
        ApiPlatform\Metadata\Post:
            securityPostDenormalize: 'is_granted("BOOK_CREATE", object)'
        ApiPlatform\Metadata\Get:
            security: 'is_granted("BOOK_READ", object)'
        ApiPlatform\Metadata\Put:
            security: 'is_granted("BOOK_EDIT", object)'
        ApiPlatform\Metadata\Delete:
            security: 'is_granted("BOOK_DELETE", object)'
```

</code-selector>

Please note that if you use both `security: "..."` and then `"post" => ["securityPostDenormalize" => "..."]`, the `security` on top level is called first, and after `securityPostDenormalize`. This could lead to unwanted behaviour, so avoid using both of them simultaneously.
If you need to use `securityPostDenormalize`, consider adding `security` for the other operations instead of the global one.

Create a *BookVoter* with the `bin/console make:voter` command:

```php
<?php
// api/src/Security/Voter/BookVoter.php

namespace App\Security\Voter;

use App\Entity\Book;
use Symfony\Component\Security\Core\Authentication\Token\TokenInterface;
use Symfony\Component\Security\Core\Authorization\Voter\Voter;
use Symfony\Component\Security\Core\Security;
use Symfony\Component\Security\Core\User\UserInterface;

class BookVoter extends Voter
{
    private $security = null;

    public function __construct(Security $security)
    {
        $this->security = $security;
    }

    protected function supports($attribute, $subject): bool
    {
        $supportsAttribute = in_array($attribute, ['BOOK_CREATE', 'BOOK_READ', 'BOOK_EDIT', 'BOOK_DELETE']);
        $supportsSubject = $subject instanceof Book;

        return $supportsAttribute && $supportsSubject;
    }

    /**
     * @param string $attribute
     * @param Book $subject
     * @param TokenInterface $token
     * @return bool
     */
    protected function voteOnAttribute($attribute, $subject, TokenInterface $token): bool
    {
        /** ... check if the user is anonymous ... **/

        switch ($attribute) {
            case 'BOOK_CREATE':
                if ( $this->security->isGranted(Role::ADMIN) ) { return true; }  // only admins can create books
                break;
            case 'BOOK_READ':
                /** ... other authorization rules ... **/
        }

        return false;
    }
}
```

*Note 1: When using Voters on POST methods: The voter needs an `$attribute` and `$subject` as input parameter, so you have to use the `securityPostDenormalize` (i.e. `"post" = { "securityPostDenormalize" = "is_granted('BOOK_CREATE', object)" }` ) because the object does not exist before denormalization (it is not created, yet.)*

*Note 2: You can't use Voters on the collection GET method, use [Collection Filters](https://api-platform.com/docs/core/security/#filtering-collection-according-to-the-current-user-permissions) instead.*

## Configuring the Access Control Error Message

By default when API requests are denied, you will get the "Access Denied" message.
You can change it by configuring the `securityMessage` attribute or the `securityPostDenormalizeMessage` attribute.

For example:

<code-selector>

```php
<?php
// api/src/Entity/Book.php
namespace App\Entity;

use ApiPlatform\Metadata\ApiResource;
use ApiPlatform\Metadata\Get;
use ApiPlatform\Metadata\Post;
use ApiPlatform\Metadata\Put;

#[ApiResource(
    security: "is_granted('ROLE_USER')",
    operations: [
        new Get(
            security: "is_granted('ROLE_USER') and object.owner == user", 
            securityMessage: 'Sorry, but you are not the book owner.'
        )
        new Put(
            securityPostDenormalize: "is_granted('ROLE_ADMIN') or (object.owner == user and previous_object.owner == user)", 
            securityPostDenormalizeMessage: 'Sorry, but you are not the actual book owner.'
        )
        new Post(
            security: "is_granted('ROLE_ADMIN')", 
            securityMessage: 'Only admins can add books.'
        )
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
        security: 'is_granted("ROLE_USER")'
        operations:
            ApiPlatform\Metadata\Post:
                security: 'is_granted("ROLE_ADMIN")'
                securityMessage: 'Only admins can add books.'
            ApiPlatform\Metadata\Get:
                security: 'is_granted("ROLE_USER") and object.owner == user'
                securityMessage: 'Sorry, but you are not the book owner.'
            ApiPlatform\Metadata\Put:
                securityPostDenormalize: "is_granted('ROLE_ADMIN') or (object.owner == user and previous_object.owner == user)"
                securityPostDenormalizeMessage: 'Sorry, but you are not the actual book owner.'
        # ...
```

</code-selector>

## Filtering Collection According to the Current User Permissions

Filtering collections according to the role or permissions of the current user must be done directly at [the state provider](state-providers.md) level. For instance, when using the built-in adapters for Doctrine ORM, MongoDB and ElasticSearch, removing entries from a collection should be done using [extensions](extensions.md).
Extensions allow to customize the generated DQL/Mongo/Elastic/... query used to retrieve the collection (e.g. add `WHERE` clauses depending of the currently connected user) instead of using access control expressions.
As extensions are services, you can [inject the Symfony `Security` class](https://symfony.com/doc/current/security.html#b-fetching-the-user-from-a-service) into them to access to current user's roles and permissions.

If you use [custom state providers](state-providers.md), you'll have to implement the filtering logic according to the persistence layer you rely on.

## Disabling Operations

To completely disable some operations from your application, refer to the [disabling operations](operations.md#enabling-and-disabling-operations)
section.

## Changing Serialization Groups Depending of the Current User

See [how to dynamically change](serialization.md#changing-the-serialization-context-dynamically) the current Serializer context according to the current logged in user.

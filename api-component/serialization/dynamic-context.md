## Change the Serialization Context Dynamically

## The User/Admin example

Let's imagine a resource where most fields can be managed by any user, but some can be managed only by admin users:

```php
<?php
// api/src/Entity/Book.php

namespace App\Entity;

use ApiPlatform\Core\Annotation\ApiResource;
use Symfony\Component\Serializer\Annotation\Groups;

/**
 * @ApiResource(
 *     normalizationContext={"groups"={"book:output"}},
 *     denormalizationContext={"groups"={"book:input"}}
 * )
 */
class Book
{
    // ...

    /**
     * This field can be managed only by an admin
     *
     * @var bool
     *
     * @Groups({"book:output", "admin:input"})
     */
    public $active = false;

    /**
     * This field can be managed by any user
     *
     * @var string
     *
     * @Groups({"book:output", "book:input"})
     */
    public $name;

    // ...
}
```

All entry points are the same for all users, so we should find a way to detect if the authenticated user is an admin, and if so
dynamically add the `admin:input` value to deserialization groups in the `$context` array.

API Platform implements a `ContextBuilder`, which prepares the context for serialization & deserialization. Let's
[decorate this service](https://symfony.com/doc/current/service_container/service_decoration.html) to override the
`createFromRequest` method:

```yaml
# api/config/services.yaml
services:
    # ...
    'App\Serializer\BookContextBuilder':
        decorates: 'api_platform.serializer.context_builder'
        arguments: [ '@App\Serializer\BookContextBuilder.inner' ]
        autoconfigure: false
```

```php
<?php
// api/src/Serializer/BookContextBuilder.php

namespace App\Serializer;

use ApiPlatform\Core\Serializer\SerializerContextBuilderInterface;
use Symfony\Component\HttpFoundation\Request;
use Symfony\Component\Security\Core\Authorization\AuthorizationCheckerInterface;
use App\Entity\Book;

final class BookContextBuilder implements SerializerContextBuilderInterface
{
    private $decorated;
    private $authorizationChecker;

    public function __construct(SerializerContextBuilderInterface $decorated, AuthorizationCheckerInterface $authorizationChecker)
    {
        $this->decorated = $decorated;
        $this->authorizationChecker = $authorizationChecker;
    }

    public function createFromRequest(Request $request, bool $normalization, ?array $extractedAttributes = null): array
    {
        $context = $this->decorated->createFromRequest($request, $normalization, $extractedAttributes);
        $resourceClass = $context['resource_class'] ?? null;

        if ($resourceClass === Book::class && isset($context['groups']) && $this->authorizationChecker->isGranted('ROLE_ADMIN') && false === $normalization) {
            $context['groups'][] = 'admin:input';
        }

        return $context;
    }
}
```

If the user has the `ROLE_ADMIN` permission and the subject is an instance of Book, `admin_input` group will be dynamically added to the
denormalization context. The `$normalization` variable lets you check whether the context is for normalization (if `TRUE`) or denormalization
(`FALSE`).

## Changing the Serialization Context on a Per-item Basis

The example above demonstrates how you can modify the normalization/denormalization context based on the current user
permissions for all books. Sometimes, however, the permissions vary depending on what book is being processed.

Think of ACL's: User "A" may retrieve Book "A" but not Book "B". In this case, we need to leverage the power of the
Symfony Serializer and register our own normalizer that adds the group on every single item (note: priority `64` is
an example; it is always important to make sure your normalizer gets loaded first, so set the priority to whatever value
is appropriate for your application; higher values are loaded earlier):

```yaml
# api/config/services.yaml
services:
    'App\Serializer\BookAttributeNormalizer':
        arguments: [ '@security.token_storage' ]
        tags:
            - { name: 'serializer.normalizer', priority: 64 }
```

The Normalizer class is a bit harder to understand, because it must ensure that it is only called once and that there is no recursion.
To accomplish this, it needs to be aware of the parent Normalizer instance itself.

Here is an example:

```php
<?php
// api/src/Serializer/BookAttributeNormalizer.php

namespace App\Serializer;

use Symfony\Component\Security\Core\Authentication\Token\Storage\TokenStorageInterface;
use Symfony\Component\Serializer\Normalizer\ContextAwareNormalizerInterface;
use Symfony\Component\Serializer\Normalizer\NormalizerAwareInterface;
use Symfony\Component\Serializer\Normalizer\NormalizerAwareTrait;
use Symfony\Component\Serializer\Normalizer\NormalizerInterface;

class BookAttributeNormalizer implements ContextAwareNormalizerInterface, NormalizerAwareInterface
{
    use NormalizerAwareTrait;

    private const ALREADY_CALLED = 'BOOK_ATTRIBUTE_NORMALIZER_ALREADY_CALLED';

    private $tokenStorage;

    public function __construct(TokenStorageInterface $tokenStorage)
    {
        $this->tokenStorage = $tokenStorage;
    }

    public function normalize($object, $format = null, array $context = [])
    {
        if ($this->userHasPermissionsForBook($object)) {
            $context['groups'][] = 'can_retrieve_book';
        }

        $context[self::ALREADY_CALLED] = true;

        return $this->normalizer->normalize($object, $format, $context);
    }

    public function supportsNormalization($data, $format = null, array $context = [])
    {
        // Make sure we're not called twice
        if (isset($context[self::ALREADY_CALLED])) {
            return false;
        }

        return $data instanceof Book;
    }

    private function userHasPermissionsForBook($object): bool
    {
        // Get permissions from user in $this->tokenStorage
        // for the current $object (book) and
        // return true or false
    }
}
```

This will add the serialization group `can_retrieve_book` only if the currently logged-in user has access to the given book
instance.

Note: In this example, we use the `TokenStorageInterface` to verify access to the book instance. However, Symfony
provides many useful other services that might be better suited to your use case. For example, the [`AuthorizationChecker`](https://symfony.com/doc/current/components/security/authorization.html#authorization-checker).

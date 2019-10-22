# Serialization Groups

You may want to restrict some resource's attributes to your GraphQL clients.

As described in the [serialization process](../serialization/index.md) documentation, you can use serialization groups to expose only the attributes you want in queries or in mutations.

If the (de)normalization context between GraphQL and REST is different, use the `graphql` key to change it.

Note that:

* A **query** is only using the normalization context.
* A **mutation** is using the denormalization context for its input and the normalization context for its output.

The following example shows you what can be done:

```php
<?php
// api/src/Entity/Book.php

namespace App\Entity;

use ApiPlatform\Core\Annotation\ApiResource;
use Symfony\Component\Serializer\Annotation\Groups;

/**
 * @ApiResource(
 *     normalizationContext={"groups"={"read"}},
 *     denormalizationContext={"groups"={"write"}},
 *     graphql={
 *         "item_query"={"normalization_context"={"groups"={"item_query"}}},
 *         "collection_query"={"normalization_context"={"groups"={"collection_query"}}},
 *         "create"={
 *             "normalization_context"={"groups"={"collection_query"}},
 *             "denormalization_context"={"groups"={"mutation"}}
 *         }
 *     }
 * )
 */
class Book
{
    // ...

    /**
     * @Groups({"read", "write", "item_query", "collection_query"})
     */
    public $name;

    /**
     * @Groups({"read", "mutation", "item_query"})
     */
    public $author;

    // ...
}
```

In this case, the REST endpoint will be able to get the two attributes of the book and to modify only its name.

The GraphQL endpoint will be able to query the name and author of an item.
It will be able to query the name of the items in the collection.
It will only be able to create a book with an author.
When doing this mutation, the author of the created book will not be returned (the name will be instead).

### Different Types when Using Different Serialization Groups

When you use different serialization groups, it will create different types in your schema.

Make sure you understand the implications when doing this: having different types means breaking the cache features in some GraphQL clients (in [Apollo Client](https://www.apollographql.com/docs/react/caching/cache-configuration/#automatic-cache-updates) for example).

For instance:
- If you use a different `normalization_context` for a mutation, a `MyResourcePayloadData` type with the restricted fields will be generated and used instead of `MyResource` (the query type).
- If you use a different `normalization_context` for the query of an item (`item_query` operation) and for the query of a collection (`collection_query` operation), two types `MyResourceItem` and `MyResourceCollection` with the restricted fields will be generated and used instead of `MyResource` (the query type).

## Changing the Serialization Context Dynamically

[As REST](../serialization/dynamic-context.md), it's possible to add dynamically a (de)serialization group when resolving a query or a mutation.

There are some differences though.

The service is `api_platform.graphql.serializer.context_builder` and the method to override is `create`.

The decorator could be like this:

```php
<?php

namespace App\Serializer;

use ApiPlatform\Core\GraphQl\Serializer\SerializerContextBuilderInterface;
use App\Entity\Book;
use Symfony\Component\Security\Core\Authorization\AuthorizationCheckerInterface;

final class BookContextBuilder implements SerializerContextBuilderInterface
{
    private $decorated;
    private $authorizationChecker;

    public function __construct(SerializerContextBuilderInterface $decorated, AuthorizationCheckerInterface $authorizationChecker)
    {
        $this->decorated = $decorated;
        $this->authorizationChecker = $authorizationChecker;
    }

    public function create(?string $resourceClass, string $operationName, array $resolverContext, bool $normalization): array
    {
        $context = $this->decorated->create($resourceClass, $operationName, $resolverContext, $normalization);
        $resourceClass = $context['resource_class'] ?? null;

        if ($resourceClass === Book::class && isset($context['groups']) && $this->authorizationChecker->isGranted('ROLE_ADMIN') && false === $normalization) {
            $context['groups'][] = 'admin:input';
        }

        return $context;
    }
}
```

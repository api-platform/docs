# Mutations

If you don't know what mutations are yet, the documentation about them is [here](https://graphql.org/learn/queries/#mutations).

For each resource, three mutations are available: one for creating it (`create`), one for updating it (`update`) and one for deleting it (`delete`).

When updating or deleting a resource, you need to pass the **IRI** of the resource as argument. See [Global Object Identifier](queries.md#global-object-identifier) for more information.

## Client Mutation Id

Following the [Relay Input Object Mutations Specification](https://facebook.github.io/relay/graphql/mutations.htm),
you can pass a `clientMutationId` as argument and can ask its value as a field.

For example, if you delete a book:

```graphql
mutation DeleteBook($id: ID!, $clientMutationId: String!) {
  deleteBook(input: {id: $id, clientMutationId: $clientMutationId}) {
    clientMutationId
  }
}
```

## Custom Mutations

Creating custom mutations is comparable to creating [custom queries](queries.md#custom-queries).

Create your resolver:

```php
<?php

namespace App\Resolver;

use ApiPlatform\Core\GraphQl\Resolver\MutationResolverInterface;
use App\Model\Book;

final class BookMutationResolver implements MutationResolverInterface
{
    /**
     * @param Book|null $item
     *
     * @return Book
     */
    public function __invoke($item, array $context)
    {
        // Mutation input arguments are in $context['args']['input'].

        // Do something with the book.
        // Or fetch the book if it has not been retrieved.

        // The returned item will pe persisted.
        return $item;
    }
}
```

As you can see, depending on how you configure your custom mutation in the resource, the item is retrieved or not.
For instance, if you don't set an `id` argument or if you disable the `read` or the `deserialize` stage (other stages [can also be disabled](resolvers-workflow.md#disabling-resolver-stages)),
the received item will be `null`.

Likewise, if you don't want your item to be persisted by API Platform,
you can return `null` instead of the mutated item (be careful: the response will also be `null`) or disable the `write` stage.

Don't forget the resolver is a service and you can inject the dependencies you want.

If you don't use autoconfiguration, add the tag `api_platform.graphql.mutation_resolver` to the resolver service.

Now in your resource:

```php
<?php

namespace App\Model;

use ApiPlatform\Core\Annotation\ApiResource;
use App\Resolver\BookMutationResolver;

/**
 * @ApiResource(graphql={
 *     "mutation"={
 *         "mutation"=BookMutationResolver::class
 *     },
 *     "withCustomArgsMutation"={
 *         "mutation"=BookMutationResolver::class,
 *         "args"={
 *             "sendMail"={"type"="Boolean!", "description"="Send a mail?"}
 *         }
 *     },
 *     "disabledStagesMutation"={
 *         "mutation"=BookMutationResolver::class,
 *         "deserialize"=false,
 *         "write"=false
 *     }
 * })
 */
class Book
{
    // ...
}
```

As the custom queries, you can define your own arguments if you don't want to use the default ones (extracted from your resource).
The only difference with them is that, even if you define your own arguments, the `clientMutationId` will always be set.

The arguments will be in `$context['args']['input']` of your resolvers.

Your custom mutations will be available like this:

```graphql
{
  mutation {
    mutationBook(input: {id: "/books/18", title: "The Fitz and the Fool"}) {
      book {
        title
      }
    }
  }

  mutation {
    withCustomArgsMutationBook(input: {sendMail: true, clientMutationId: "myId}) {
      book {
        title
      }
      clientMutationId
    }
  }

  mutation {
    disabledStagesMutationBook(input: {id: "/books/18", title: "The Fitz and the Fool"}) {
      book {
        title
      }
      clientMutationId
    }
  }
}
```

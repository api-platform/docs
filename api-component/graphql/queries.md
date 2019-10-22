# Queries

If you don't know what queries are yet, the documentation about them is [here](https://graphql.org/learn/queries/).

For each resource, two queries are available: one for retrieving an item and the other one for the collection.
For example, if you have a `Book` resource, the queries `book` and `books` can be used.

## Global Object Identifier

When querying an item, you need to pass an identifier as argument. Following the [Relay Global Object Identification Specification](https://facebook.github.io/relay/graphql/objectidentification.htm),
the identifier needs to be globally unique. In API Platform, this argument is represented as an [IRI (Internationalized Resource Identifier)](https://www.w3.org/TR/ld-glossary/#internationalized-resource-identifier).

For example, to query a book having as identifier `89`, you have to run the following:

```graphql
{
  book(id: "/books/89") {
    title
    isbn
  }
}
```

Note that in this example, we're retrieving two fields: `title` and `isbn`.

## Custom Queries

To create a custom query, first of all you need to create its resolver.

If you want a custom query for a collection, create a class like this:

```php
<?php

namespace App\Resolver;

use ApiPlatform\Core\GraphQl\Resolver\QueryCollectionResolverInterface;
use App\Model\Book;

final class BookCollectionResolver implements QueryCollectionResolverInterface
{
    /**
     * @param iterable<Book> $collection
     *
     * @return iterable<Book>
     */
    public function __invoke(iterable $collection, array $context): iterable
    {
        // Query arguments are in $context['args'].

        foreach ($collection as $book) {
            // Do something with the book.
        }

        return $collection;
    }
}
```

If you use autoconfiguration (the default Symfony configuration) in your application, then you are done!

Else, you need to tag your resolver like this:

```yaml
# api/config/services.yaml
services:
    # ...
    App\Resolver\BookCollectionResolver:
        tags:
            - { name: api_platform.graphql.query_resolver }
```

The resolver for an item is very similar:

```php
<?php

namespace App\Resolver;

use ApiPlatform\Core\GraphQl\Resolver\QueryItemResolverInterface;
use App\Model\Book;

final class BookResolver implements QueryItemResolverInterface
{
    /**
     * @param Book|null $item
     *
     * @return Book
     */
    public function __invoke($item, array $context)
    {
        // Query arguments are in $context['args'].

        // Do something with the book.
        // Or fetch the book if it has not been retrieved.

        return $item;
    }
}
```

Note that you will receive the retrieved item or not in this resolver depending on how you configure your query in your resource.

Since the resolver is a service, you can inject some dependencies and fetch your item in the resolver if you want.

If you don't use autoconfiguration, don't forget to tag your resolver with `api_platform.graphql.query_resolver`.

Now that your resolver is created and registered, you can configure your custom query and link its resolver.

In your resource, add the following:

```php
<?php

namespace App\Model;

use ApiPlatform\Core\Annotation\ApiResource;
use App\Resolver\BookCollectionResolver;
use App\Resolver\BookResolver;

/**
 * @ApiResource(graphql={
 *     "retrievedQuery"={
 *         "item_query"=BookResolver::class
 *     },
 *     "notRetrievedQuery"={
 *         "item_query"=BookResolver::class,
 *         "args"={}
 *     },
 *     "withDefaultArgsNotRetrievedQuery"={
 *         "item_query"=BookResolver::class,
 *         "read"=false
 *     },
 *     "withCustomArgsQuery"={
 *         "item_query"=BookResolver::class,
 *         "args"={
 *             "id"={"type"="ID!"},
 *             "log"={"type"="Boolean!", "description"="Is logging activated?"},
 *             "logDate"={"type"="DateTime"}
 *         }
 *     },
 *     "collectionQuery"={
 *         "collection_query"=BookCollectionResolver::class
 *     }
 * })
 */
class Book
{
    // ...
}
```

As you can see, it's possible to define your own arguments for your custom queries.
They are following the GraphQL type system.
If you don't define the `args` property, it will be the default ones (for example `id` for an item).

If you don't want API Platform to retrieve the item for you, disable the `read` stage like in `withDefaultArgsNotRetrievedQuery`.
Some other stages [can be disabled](resolvers-workflow.md#disabling-resolver-stages).
Another option would be to make sure there is no `id` argument.
This is the case for `notRetrievedQuery` (empty args).
Conversely, if you need to add custom arguments, make sure `id` is added among the arguments if you need the item to be retrieved automatically.

Note also that:
- If you have added your [own custom types](custom-types.md), you can use them directly for your arguments types (it's the case here for `DateTime`).
- You can also add a custom description for your custom arguments. You can see the [field arguments documentation](https://webonyx.github.io/graphql-php/type-system/object-types/#field-arguments) for more options.

The arguments you have defined or the default ones and their value will be in `$context['args']` of your resolvers.

You custom queries will be available like this:

```graphql
{
  retrievedQueryBook(id: "/books/56") {
    title
  }

  notRetrievedQueryBook {
    title
  }

  withDefaultArgsNotRetrievedQueryBook(id: "/books/56") {
    title
  }

  withCustomArgsQueryBook(id: "/books/23", log: true, logDate: "2019-12-20") {
    title
  }

  collectionQueryBooks {
    edges {
      node {
        title
      }
    }
  }
}
```

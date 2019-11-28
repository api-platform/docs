# GraphQL Support

[GraphQL](https://graphql.org/) is a query language made to communicate with an API and therefore is an alternative to REST.

It has some advantages compared to REST: it solves the over-fetching or under-fetching of data, is strongly typed, and is capable of retrieving multiple and nested data in one go, but it also comes with drawbacks. For example it creates overhead depending on the request.

API Platform creates a REST API by default. But you can choose to enable GraphQL as well.

Once enabled, you have nothing to do: your schema describing your API is automatically built and your GraphQL endpoint is ready to go!

## Enabling GraphQL

To enable GraphQL and its IDE (GraphiQL and GraphQL Playground) in your API, simply require the [graphql-php](https://webonyx.github.io/graphql-php/) package using Composer and clear the cache one more time:

    $ docker-compose exec php composer req webonyx/graphql-php && docker-compose exec php bin/console cache:clear

You can now use GraphQL at the endpoint: `https://localhost:8443/graphql`.

*Note:* If you used [Symfony Flex to install API Platform](../distribution/index.md#using-symfony-flex-and-composer-advanced-users),
the GraphQL endpoint will be: `https://localhost:8443/api/graphql`.

## GraphiQL

If Twig is installed in your project, go to the GraphQL endpoint with your browser. You will see a nice interface provided by GraphiQL to interact with your API.

The GraphiQL IDE can also be found at `/graphql/graphiql`.

If you need to disable it, it can be done in the configuration:

```yaml
# api/config/packages/api_platform.yaml
api_platform:
    graphql:
        graphiql:
            enabled: false
# ...
```

### Add another Location for GraphiQL

If you want to add a different location besides `/graphql/graphiql`, you can do it like this:

```yaml
# app/config/routes.yaml
graphiql:
    path: /docs/graphiql
    controller: api_platform.graphql.action.graphiql
```

## GraphQL Playground

Another IDE is by default included in API Platform: GraphQL Playground.

It can be found at `/graphql/graphql_playground`.

You can disable it if you want in the configuration:

```yaml
# api/config/packages/api_platform.yaml
api_platform:
    graphql:
        graphql_playground:
            enabled: false
# ...
```

### Add another Location for GraphQL Playground

You can add a different location besides `/graphql/graphql_playground`:

```yaml
# app/config/routes.yaml
graphql_playground:
    path: /docs/graphql_playground
    controller: api_platform.graphql.action.graphql_playground
```

## Modifying or Disabling the Default IDE

When going to the GraphQL endpoint, you can choose to launch the IDE you want.

```yaml
# api/config/packages/api_platform.yaml
api_platform:
    graphql:
        # Choose between graphiql or graphql-playground
        default_ide: graphql-playground
# ...
```

You can also disable this feature by setting the configuration value to `false`.

```yaml
# api/config/packages/api_platform.yaml
api_platform:
    graphql:
        default_ide: false
# ...
```

## Request with `application/graphql` Content-Type

If you wish to send a [POST request using the `application/graphql` Content-Type](https://graphql.org/learn/serving-over-http/#post-request),
you need to enable it in the [allowed formats of API Platform](content-negotiation.md#configuring-formats-globally):

```yaml
# api/config/packages/api_platform.yaml
api_platform:
    formats:
        # ...
        graphql: ['application/graphql']
```

## Queries

If you don't know what queries are yet, please [read the documentation about them](https://graphql.org/learn/queries/).

For each resource, two queries are available: one for retrieving an item and the other one for the collection.
For example, if you have a `Book` resource, the queries `book` and `books` can be used.

### Global Object Identifier

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

### Custom Queries

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
Some other stages [can be disabled](#disabling-resolver-stages).
Another option would be to make sure there is no `id` argument.
This is the case for `notRetrievedQuery` (empty args).
Conversely, if you need to add custom arguments, make sure `id` is added among the arguments if you need the item to be retrieved automatically.

Note also that:
- If you have added your [own custom types](#custom-types), you can use them directly for your arguments types (it's the case here for `DateTime`).
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

## Mutations

If you don't know what mutations are yet, the documentation about them is [here](https://graphql.org/learn/queries/#mutations).

For each resource, three mutations are available: one for creating it (`create`), one for updating it (`update`) and one for deleting it (`delete`).

When updating or deleting a resource, you need to pass the **IRI** of the resource as argument. See [Global Object Identifier](#global-object-identifier) for more information.

### Client Mutation Id

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

### Custom Mutations

Creating custom mutations is comparable to creating [custom queries](#custom-queries).

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
For instance, if you don't set an `id` argument or if you disable the `read` or the `deserialize` stage (other stages [can also be disabled](#disabling-resolver-stages)),
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

## Workflow of the Resolvers

API Platform resolves the queries and mutations by using its own **resolvers**.

Even if you create your [custom queries](#custom-queries) or your [custom mutations](#custom-mutations),
these resolvers will be used and yours will be called at the right time.

Each resolver follows a workflow composed of **stages**.

The schema below describes them:

![Resolvers Workflow](images/diagrams/resolvers-workflow.svg)

Each stage corresponds to a service. It means you can take control of the workflow wherever you want by decorating them!

Here is an example of the decoration of the write stage, for instance if you want to persist your data as you want.

Create your *WriteStage*:

```php
<?php

namespace App\Stage;

use ApiPlatform\Core\GraphQl\Resolver\Stage\WriteStageInterface;

final class WriteStage implements WriteStageInterface
{
    private $writeStage;

    public function __construct(WriteStageInterface $writeStage)
    {
        $this->writeStage = $writeStage;
    }

    /**
     * {@inheritdoc}
     */
    public function __invoke($data, string $resourceClass, string $operationName, array $context)
    {
        // You can add pre-write code here.

        // Call the decorated write stage (this syntax calls the __invoke method).
        $writtenObject = ($this->writeStage)($data, $resourceClass, $operationName, $context);

        // You can add post-write code here.

        return $writtenObject;
    }
}
```

Decorate the API Platform stage service:

```yaml
# api/config/services.yaml
services:
    # ...
    'App\Stage\WriteStage':
        decorates: api_platform.graphql.resolver.stage.write
```

### Disabling Resolver Stages

If you need to, you can disable some stages done by the resolvers, for instance if you don't want your data to be validated.

The following table lists the stages you can disable in your resource configuration.

Attribute     | Type   | Default | Description
--------------|--------|---------|-------------
`read`        | `bool` | `true`  | Enables or disables the reading of data
`deserialize` | `bool` | `true`  | Enables or disables the deserialization of data (mutation only)
`validate`    | `bool` | `true`  | Enables or disables the validation of the denormalized data (mutation only)
`write`       | `bool` | `true`  | Enables or disables the writing of data into the persistence system (mutation only)
`serialize`   | `bool` | `true`  | Enables or disables the serialization of data

A stage can be disabled at the operation level:

```php
<?php

namespace App\Model;

use ApiPlatform\Core\Annotation\ApiResource;

/**
 * @ApiResource(graphql={
 *     "mutation"={
 *         "write"=false
 *     }
 * })
 */
class Book
{
    // ...
}
```

Or at the resource attributes level (will be also applied in REST and for all operations):

```php
<?php

namespace App\Model;

use ApiPlatform\Core\Annotation\ApiResource;

/**
 * @ApiResource(
 *     graphql={...},
 *     attributes={
 *         "write"=false
 *     }
 * })
 */
class Book
{
    // ...
}
```

## Events

No events are sent by the resolvers in API Platform. If you want to add your custom logic, [decorating the stages](#workflow-of-the-resolvers) is
the recommended way to do it.

However, if you really want to use events, you can by installing a [bundle dispatching events before and after the stages](https://github.com/alanpoulain/ApiPlatformEventsBundle).

## Filters

Filters are supported out-of-the-box. Follow the [filters](filters.md) documentation and your filters will be available as arguments of queries.

However you don't necessarily have the same needs for your GraphQL endpoint as for your REST one.

In the `ApiResource` declaration, you can choose to decorrelate the GraphQL filters in `collection_query` of the `graphql` attribute.
In order to keep the default behavior (possibility to fetch, delete, update or create), define all the operations (`item_query` ,`collection_query` , `delete`, `update` and `create`).

For example, this entity will have a search filter for REST and a date filter for GraphQL:

```php
<?php
// api/src/Entity/Offer.php

namespace App\Entity;

use ApiPlatform\Core\Annotation\ApiResource;

/**
 * @ApiResource(
 *     attributes={
 *         "filters"={"offer.search_filter"}
 *     },
 *     graphql={
 *         "item_query",
 *         "collection_query"={
 *              "filters"={"offer.date_filter"}
 *          },
 *          "delete",
 *          "update",
 *          "create"
 *     }
 * )
 */
class Offer
{
    // ...
}
```

### Filtering on Nested Properties

Unlike for REST, all built-in filters support nested properties using the underscore (`_`) syntax instead of the dot (`.`) syntax, e.g.:

```php
<?php
// api/src/Entity/Offer.php

namespace App\Entity;

use ApiPlatform\Core\Annotation\ApiFilter;
use ApiPlatform\Core\Annotation\ApiResource;
use ApiPlatform\Core\Bridge\Doctrine\Orm\Filter\OrderFilter;
use ApiPlatform\Core\Bridge\Doctrine\Orm\Filter\SearchFilter;

/**
 * @ApiResource
 * @ApiFilter(OrderFilter::class, properties={"product.releaseDate"})
 * @ApiFilter(SearchFilter::class, properties={"product.color": "exact"})
 */
class Offer
{
    // ...
}
```

The above allows you to find offers by their respective product's color like for the REST Api.
You can then filter using the following syntax:

```graphql
{
  offers(product_color: "red") {
    edges {
      node {
        id
        product {
          name
          color
        }
      }
    }
  }
}
```

Or order your results like:

```graphql
{
  offers(order: {product_releaseDate: "DESC"}) {
    edges {
      node {
        id
        product {
          name
          color
        }
      }
    }
  }
}
```
Another difference with the REST API filters is that the keyword `_list` must be used instead of the traditional `[]` to filter over multiple values.

For example, if you want to search the offers with a green or a red product you can use the following syntax:
```graphql
{
  offers(product_color_list: ["red", "green"]) {
    edges {
      node {
        id
        product {
          name
          color
        }
      }
    }
  }
}
```

## Pagination

API Platform natively enables a cursor-based pagination for collections.
It supports [GraphQL's Complete Connection Model](https://graphql.org/learn/pagination/#complete-connection-model) and is compatible with [Relay's Cursor Connections Specification](https://facebook.github.io/relay/graphql/connections.htm).

### Using the Cursor-based Pagination

Here is an example query leveraging the pagination system:

```graphql
{
  offers(first: 10, after: "cursor") {
    totalCount
    pageInfo {
      endCursor
      hasNextPage
    }
    edges {
      cursor
      node {
        id
      }
    }
  }
}
```

Two pairs of parameters work with the query:

* `first` and `after`;
* `last` and `before`.

More precisely:

* `first` corresponds to the items per page starting from the beginning;
* `after` corresponds to the `cursor` from which the items are returned.

* `last` corresponds to the items per page starting from the end;
* `before` corresponds to the `cursor` from which the items are returned, from a backwards point of view.

The current page always has a `startCursor` and an `endCursor`, present in the `pageInfo` field.

To get the next page, you would add the `endCursor` from the current page as the `after` parameter.

```graphql
{
  offers(first: 10, after: "endCursor") {
  }
}
```

For the previous page, you would add the `startCursor` from the current page as the `before` parameter.

```graphql
{
  offers(last: 10, before: "startCursor") {
  }
}
```

How do you know when you have reached the last page? It is the aim of the property `hasNextPage` or `hasPreviousPage` in `pageInfo`.
When it is false, you know it is the last page and moving forward or backward will give you an empty result.

### Disabling the Pagination

See also the [pagination documentation](pagination.md#disabling-the-pagination).

#### Globally

The pagination can be disabled for all GraphQL resources using this configuration:

```yaml
# api/config/packages/api_platform.yaml
api_platform:
    graphql:
        collection:
            pagination:
                enabled: false
```

#### For a Specific Resource

It can also be disabled for a specific resource (REST and GraphQL):

```php
<?php
// api/src/Entity/Book.php

use ApiPlatform\Core\Annotation\ApiResource;

/**
 * @ApiResource(attributes={"pagination_enabled"=false})
 */
class Book
{
    // ...
}
```

#### For a Specific Resource Collection Operation

You can also disable the pagination for a specific collection operation:

```php
<?php
// api/src/Entity/Book.php

use ApiPlatform\Core\Annotation\ApiResource;

/**
 * @ApiResource(graphql={"collection_query"={"pagination_enabled"=false}})
 */
class Book
{
    // ...
}
```

## Security

To add a security layer to your queries and mutations, follow the [security](security.md) documentation.

If your security needs differ between REST and GraphQL, add the particular parts in the `graphql` key.

In the example below, we want the same security rules as we have in REST, but we also want to allow an admin to delete a book only in GraphQL.
Please note that, it's not possible to update a book in GraphQL because the `update` operation is not defined.

```php
<?php
// api/src/Entity/Book.php

namespace App\Entity;

use ApiPlatform\Core\Annotation\ApiResource;

/**
 * @ApiResource(
 *     attributes={"security"="is_granted('ROLE_USER')"},
 *     collectionOperations={
 *         "post"={"security"="is_granted('ROLE_ADMIN')", "security_message"="Only admins can add books."}
 *     },
 *     itemOperations={
 *         "get"={"security"="is_granted('ROLE_USER') and object.owner == user", "security_message"="Sorry, but you are not the book owner."}
 *     },
 *     graphql={
 *         "item_query"={"security"="is_granted('ROLE_USER') and object.owner == user"},
 *         "collection_query"={"security"="is_granted('ROLE_ADMIN')"},
 *         "delete"={"security"="is_granted('ROLE_ADMIN')"},
 *         "create"={"security"="is_granted('ROLE_ADMIN')"}
 *     }
 * )
 */
class Book
{
    // ...
}
```

## Serialization Groups

You may want to restrict some resource's attributes to your GraphQL clients.

As described in the [serialization process](serialization.md) documentation, you can use serialization groups to expose only the attributes you want in queries or in mutations.

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

## Name Conversion

You can modify how the property names of your resources are converted into field and filter names of your GraphQL schema.

By default the property name will be used without conversion. If you want to apply a name converter, follow the [Name Conversion documentation](serialization.md#name-conversion).

For instance, your resource can have properties in camelCase:

```php
<?php
// api/src/Entity/Book.php

namespace App\Entity;

use ApiPlatform\Core\Annotation\ApiResource;
use ApiPlatform\Core\Annotation\ApiFilter;
use ApiPlatform\Core\Bridge\Doctrine\Orm\Filter\SearchFilter;

/**
 * @ApiResource
 * @ApiFilter(SearchFilter::class, properties={"publicationDate": "partial"})
 */
class Book
{
    // ...

    public $publicationDate;

    // ...
}
```

By default, with the search filter, the query to retrieve a collection will be:

```graphql
{
  books(publicationDate: "2010") {
    edges {
      node {
        publicationDate
      }
    }
  }
}
```

But if you use the `CamelCaseToSnakeCaseNameConverter`, it will be:

```graphql
{
  books(publication_date: "2010") {
    edges {
      node {
        publication_date
      }
    }
  }
}
```

### Nesting Separator

If you use snake_case, you can wonder how to make the difference between an underscore and the separator of the nested fields in the filter names, by default an underscore too.

For instance if you have this resource:

```php
<?php
// api/src/Entity/Book.php

namespace App\Entity;

use ApiPlatform\Core\Annotation\ApiResource;
use ApiPlatform\Core\Annotation\ApiFilter;
use ApiPlatform\Core\Bridge\Doctrine\Orm\Filter\SearchFilter;

/**
 * @ApiResource
 * @ApiFilter(SearchFilter::class, properties={"relatedBooks.name": "exact"})
 */
class Book
{
    // ...

    public $name;

    /**
     * @ORM\OneToMany(targetEntity="Book")
     */
    public $relatedBooks;

    // ...
}
```

You would need to use the search filter like this:

```graphql
{
  books(related_books_name: "The Fitz and the Fool") {
    edges {
      node {
        name
      }
    }
  }
}
```

To avoid this issue, you can configure the nesting separator to use, for example, `__` instead of `_`:

```yaml
# api/config/packages/api_platform.yaml
api_platform:
    graphql:
        nesting_separator: __
# ...
```

In this case, your query will be:

```graphql
{
  books(related_books__name: "The Fitz and the Fool") {
    edges {
      node {
        name
      }
    }
  }
}
```

Much better, isn't it?

## Custom Types

You might need to add your own types to your GraphQL application.

Create your type class by implementing the interface `ApiPlatform\Core\GraphQl\Type\Definition\TypeInterface`.

You should extend the `GraphQL\Type\Definition\ScalarType` class too to take advantage of its useful methods.

For instance, to create a custom `DateType`:

```php
<?php

namespace App\Type\Definition;

use ApiPlatform\Core\GraphQl\Type\Definition\TypeInterface;
use GraphQL\Error\Error;
use GraphQL\Language\AST\StringValueNode;
use GraphQL\Type\Definition\ScalarType;
use GraphQL\Utils\Utils;

final class DateTimeType extends ScalarType implements TypeInterface
{
    public function __construct()
    {
        $this->name = 'DateTime';
        $this->description = 'The `DateTime` scalar type represents time data.';

        parent::__construct();
    }

    public function getName(): string
    {
        return $this->name;
    }

    /**
     * {@inheritdoc}
     */
    public function serialize($value)
    {
        // Already serialized.
        if (\is_string($value)) {
            return (new \DateTime($value))->format('Y-m-d');
        }

        if (!($value instanceof \DateTime)) {
            throw new Error(sprintf('Value must be an instance of DateTime to be represented by DateTime: %s', Utils::printSafe($value)));
        }

        return $value->format(\DateTime::ATOM);
    }

    /**
     * {@inheritdoc}
     */
    public function parseValue($value)
    {
        if (!\is_string($value)) {
            throw new Error(sprintf('DateTime cannot represent non string value: %s', Utils::printSafeJson($value)));
        }

        if (false === \DateTime::createFromFormat(\DateTime::ATOM, $value)) {
            throw new Error(sprintf('DateTime cannot represent non date value: %s', Utils::printSafeJson($value)));
        }

        // Will be denormalized into a \DateTime.
        return $value;
    }

    /**
     * {@inheritdoc}
     */
    public function parseLiteral($valueNode, ?array $variables = null)
    {
        if ($valueNode instanceof StringValueNode && false !== \DateTime::createFromFormat(\DateTime::ATOM, $valueNode->value)) {
            return $valueNode->value;
        }

        // Intentionally without message, as all information already in wrapped Exception
        throw new \Exception();
    }
}
```

You can also check the documentation of [graphql-php](https://webonyx.github.io/graphql-php/type-system/scalar-types/#writing-custom-scalar-types).

The big difference in API Platform is that the value is already serialized when it's received in your type class.
Similarly, you would not want to denormalize your parsed value since it will be done by API Platform later.

If you use autoconfiguration (the default Symfony configuration) in your application, then you are done!

Else, you need to tag your type class like this:

```yaml
# api/config/services.yaml
services:
    # ...
    App\Type\Definition\DateTimeType:
        tags:
            - { name: api_platform.graphql.type }
```

Your custom type is now registered and is available in the `TypesContainer`.

To use it please [modify the extracted types](#modify-the-extracted-types) or use it directly in [custom queries](#custom-queries) or [custom mutations](#custom-mutations).

## Modify the Extracted Types

The GraphQL schema and its types are extracted from your resources.
In some cases, you would want to modify the extracted types for instance to use your custom ones.

To do so, you need to decorate the `api_platform.graphql.type_converter` service:

```yaml
# api/config/services.yaml
services:
    # ...
    'App\Type\TypeConverter':
        decorates: api_platform.graphql.type_converter
```

Your class needs to look like this:

```php
<?php

namespace App\Type;

use ApiPlatform\Core\GraphQl\Type\TypeConverterInterface;
use App\Model\Book;
use Symfony\Component\PropertyInfo\Type;

final class TypeConverter implements TypeConverterInterface
{
    private $defaultTypeConverter;

    public function __construct(TypeConverterInterface $defaultTypeConverter)
    {
        $this->defaultTypeConverter = $defaultTypeConverter;
    }

    /**
     * {@inheritdoc}
     */
    public function convertType(Type $type, bool $input, ?string $queryName, ?string $mutationName, string $resourceClass, string $rootResource, ?string $property, int $depth)
    {
        if ('publicationDate' === $property
            && Book::class === $resourceClass
        ) {
            return 'DateTime';
        }

        return $this->defaultTypeConverter->convertType($type, $input, $queryName, $mutationName, $resourceClass, $rootResource, $property, $depth);
    }
}
```

In this case, the `publicationDate` property of the `Book` class will have a custom `DateTime` type.

You can even apply this logic for a kind of property. Replace the previous condition with something like this:

```php
if (Type::BUILTIN_TYPE_OBJECT === $type->getBuiltinType()
    && is_a($type->getClassName(), \DateTimeInterface::class, true)
) {
    return 'DateTime';
}
```

All `DateTimeInterface` properties will have the `DateTime` type in this example.

## Changing the Serialization Context Dynamically

[As REST](serialization.md#changing-the-serialization-context-dynamically), it's possible to add dynamically a (de)serialization group when resolving a query or a mutation.

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

## Export the Schema in SDL

You may need to export your schema in SDL (Schema Definition Language) to import it in some tools.

The `api:graphql:export` command is provided to do so:

```bash
docker-compose exec php bin/console api:graphql:export -o path/to/your/volume/schema.graphql
```

Since the command prints the schema to the output if you don't use the `-o` option, you can also use this command:

```bash
docker-compose exec php bin/console api:graphql:export > path/in/host/schema.graphql
```

## Handling File Upload

Please follow the [file upload documentation](file-upload.md), only the differences will be documented here.

The file upload with GraphQL follows the [GraphQL multipart request specification](https://github.com/jaydenseric/graphql-multipart-request-spec).

You can also upload multiple files at the same time.

### Configuring the Entity Receiving the Uploaded File

Configure the entity by adding a [custom mutation resolver](#custom-mutations):

```php
<?php
// api/src/Entity/MediaObject.php

namespace App\Entity;

use ApiPlatform\Core\Annotation\ApiProperty;
use ApiPlatform\Core\Annotation\ApiResource;
use App\Resolver\CreateMediaObjectResolver;
use Doctrine\ORM\Mapping as ORM;
use Symfony\Component\HttpFoundation\File\File;
use Symfony\Component\Serializer\Annotation\Groups;
use Symfony\Component\Validator\Constraints as Assert;
use Vich\UploaderBundle\Mapping\Annotation as Vich;

/**
 * @ORM\Entity
 * @ApiResource(
 *     iri="http://schema.org/MediaObject",
 *     normalizationContext={
 *         "groups"={"media_object_read"}
 *     },
 *     graphql={
 *         "upload"={
 *             "mutation"=CreateMediaObjectResolver::class,
 *             "deserialize"=false,
 *             "args"={
 *                 "file"={"type"="Upload!", "description"="The file to upload"}
 *             }
 *         }
 *     }
 * )
 * @Vich\Uploadable
 */
class MediaObject
{
    /**
     * @var int|null
     *
     * @ORM\Column(type="integer")
     * @ORM\GeneratedValue
     * @ORM\Id
     */
    protected $id;

    /**
     * @var string|null
     *
     * @ApiProperty(iri="http://schema.org/contentUrl")
     * @Groups({"media_object_read"})
     */
    public $contentUrl;

    /**
     * @var File|null
     *
     * @Assert\NotNull(groups={"media_object_create"})
     * @Vich\UploadableField(mapping="media_object", fileNameProperty="filePath")
     */
    public $file;

    /**
     * @var string|null
     *
     * @ORM\Column(nullable=true)
     */
    public $filePath;

    public function getId(): ?int
    {
        return $this->id;
    }
}
```

As you can see, a dedicated type `Upload` is used in the argument of the `upload` mutation.

If you need to upload multiple files, replace `"file"={"type"="Upload!", "description"="The file to upload"}`
with `"files"={"type"="[Upload!]!", "description"="Files to upload"}`.

You don't need to create it, it's provided in API Platform.

### Resolving the File Upload

The corresponding resolver you added in the resource configuration should be written like this:

```php
<?php
// api/src/Resolver/CreateMediaObjectResolver.php

namespace App\Resolver;

use ApiPlatform\Core\GraphQl\Resolver\MutationResolverInterface;
use App\Entity\MediaObject;
use Symfony\Component\HttpFoundation\File\UploadedFile;
use Vich\UploaderBundle\Storage\StorageInterface;

final class CreateMediaObjectResolver implements MutationResolverInterface
{
    /**
     * @var StorageInterface
     */
    private $storage;

    public function __construct(StorageInterface $storage)
    {
        $this->storage = $storage;
    }

    /**
     * @param null $item
     */
    public function __invoke($item, array $context): MediaObject
    {
        $uploadedFile = $context['args']['input']['file'];

        $mediaObject = new MediaObject();
        $mediaObject->file = $uploadedFile;
        $mediaObject->contentUrl = $this->storage->resolveUri($mediaObject); 
        return $mediaObject;
    }
}
```

For handling the upload of multiple files, iterate over `$context['args']['input']['files']`.

### Using the `createMediaObject` Mutation

Following the specification, the upload must be done with a `multipart/form-data` content type.

You need to enable it in the [allowed formats of API Platform](content-negotiation.md#configuring-formats-globally):

```yaml
# api/config/packages/api_platform.yaml
api_platform:
    formats:
        # ...
        multipart: ['multipart/form-data']
```

You can now upload files using the `createMediaObject` mutation, for details check [GraphQL multipart request specification](https://github.com/jaydenseric/graphql-multipart-request-spec)
and for an example implementation for the Apollo client check out [Apollo Upload Client](https://github.com/jaydenseric/apollo-upload-client).

```graphql
mutation CreateMediaObject($file: Upload!) {
    createMediaObject(input: {file: $file}) {
        mediaObject {
            id
            contentUrl
        }
    }
}
```

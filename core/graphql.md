# GraphQL Support

## Overall View

[GraphQL](http://graphql.org/) is a query language made to communicate with an API and therefore is an alternative to REST.

It has some advantages compared to REST: it solves the over-fetching or under-fetching of data, is strongly typed, and is capable of retrieving multiple and nested data in one go, but it also comes with drawbacks. For example it creates overhead depending on the request.

API Platform creates a REST API by default. But you can choose to enable GraphQL as well.

Once enabled, you have nothing to do: your schema describing your API is automatically built and your GraphQL endpoint is ready to go!

## Preface
Because GraphQL is strongly typed it is important to define type of your properties and insure that they are readable.
API Platform is using [`ApiPlatform\Core\GraphQl\Type\SchemaBuilder`](https://github.com/api-platform/core/blob/8f0b897c6d12c11698b7d37f41c42fe6c99e1882/src/GraphQl/Type/SchemaBuilder.php#L47) to build schema for GraphQL

API Platform is using `PropertyInfo` component which will try to extract field type using: ORM, PhpDoc and Reflection

## Enabling GraphQL

To enable GraphQL and GraphiQL interface in your API, simply require the [graphql-php](https://webonyx.github.io/graphql-php/) package using Composer and clear the cache one more time:

```bash
docker-compose exec php composer req webonyx/graphql-php && bin/console cache:clear
```

You can now use GraphQL at the endpoint: `https://localhost:8443/graphql`.

*Note:* If you used [Symfony Flex to install API Platform](../distribution/index.md#using-symfony-flex-and-composer-advanced-users),
the GraphQL endpoint will be: `https://localhost:8443/api/graphql`.

## GraphiQL

If Twig is installed in your project, go to the GraphQL endpoint with your browser. You will see a nice interface provided by GraphiQL to interact with your API.

If you need to disable it, it can be done in the configuration:

```yaml
# api/config/packages/api_platform.yaml
api_platform:
    graphql:
        graphiql:
            enabled: false
# ...            
```

## Queries

If you don't know what queries are yet, the documentation about them is [here](https://graphql.org/learn/queries/).

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

### Pagination

API Platform natively enables a cursor-based pagination for collections.
It supports [GraphQL's Complete Connection Model](https://graphql.org/learn/pagination/#complete-connection-model) and is compatible with [Relay's Cursor Connections Specification](https://facebook.github.io/relay/graphql/connections.htm).

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

The two parameters `first` and `after` are necessary to make the paginated query work, more precisely:

* `first` corresponds to the items per page starting from the beginning;
* `after` corresponds to the `cursor` from which the items are returned.

The current page always has an `endCursor` present in the `pageInfo` field.
To get the next page, you would add the `endCursor` from the current page as the `after` parameter.

```graphql
{
  offers(first: 10, after: "endCursor") {
  }
}
```

When the property `hasNextPage` of the `pageInfo` field is false, you've reached the last page.
If you move forward, you'll end up having an empty result.

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

## Filters

Filters are supported out-of-the-box. Follow the [filters](filters.md) documentation and your filters will be available as arguments of queries.

However you don't necessarily have the same needs for your GraphQL endpoint as for your REST one.

In the `ApiResource` declaration, you can choose to decorrelate the GraphQL filters in `query` of the `graphql` attribute.
In order to keep the default behavior (possibility to fetch, delete, update or create), define all the operations (`query`, `delete`, `update` and `create`).

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
 *         "query"={
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

## Security (`access_control`)

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
 *     attributes={"access_control"="is_granted('ROLE_USER')"},
 *     collectionOperations={
 *         "post"={"access_control"="is_granted('ROLE_ADMIN')", "access_control_message"="Only admins can add books."}
 *     },
 *     itemOperations={
 *         "get"={"access_control"="is_granted('ROLE_USER') and object.owner == user", "access_control_message"="Sorry, but you are not the book owner."}
 *     },
 *     graphql={
 *         "query"={"access_control"="is_granted('ROLE_USER') and object.owner == user"},
 *         "delete"={"access_control"="is_granted('ROLE_ADMIN')"},
 *         "create"={"access_control"="is_granted('ROLE_ADMIN')"}
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
 *         "query"={"normalization_context"={"groups"={"query"}}},
 *         "create"={
 *             "normalization_context"={"groups"={"query"}},
 *             "denormalization_context"={"groups"={"mutation"}}
 *         }
 *     }
 * )
 */
class Book
{
    // ...

    /**
     * @Groups({"read", "write", "query"})
     */
    public $name;

    /**
     * @Groups({"read", "mutation"})
     */
    public $author;

    // ...
}
```

In this case, the REST endpoint will be able to get the two attributes of the book and to modify only its name.

The GraphQL endpoint will be able to query only the name. It will only be able to create a book with an author.
When doing this mutation, the author of the created book will not be returned (the name will be instead).

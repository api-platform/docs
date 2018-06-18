# GraphQL Support

## Overall View

[GraphQL](http://graphql.org/) is a query language made to communicate with an API and therefore is an alternative to REST.

It has some advantages compared to REST: it solves the over-fetching or under-fetching of data, is strongly typed, and is capable of retrieving multiple and nested data in one time; but it also comes with drawbacks: for example it creates overhead depending of the request.

API Platform creates a REST API by default. But you can choose to enable GraphQL as well.

Once enabled, you have nothing to do: your schema describing your API is automatically built and your GraphQL endpoint is ready to go!

## Enabling GraphQL

To enable GraphQL and GraphiQL interface in your API, simply require the graphql-php package using Composer:

    $ composer require webonyx/graphql-php

You can now use GraphQL at the endpoint: `http://localhost/graphql`.

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
 *     attributes={
 *         "normalization_context"={"groups"={"read"}},
 *         "denormalization_context"={"groups"={"write"}}
 *     },
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
    private $name;

    /**
     * @Groups({"read", "mutation"})
     */
    private $author;

    // ...
}
```

In this case, the REST endpoint will be able to get the two attributes of the book and to modify only its name.

The GraphQL endpoint will be able to query only the name. It will only be able to create a book with an author.
When doing this mutation, the author of the created book will not be returned (the name will be instead).

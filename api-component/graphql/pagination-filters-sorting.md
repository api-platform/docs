# Pagination, Filters & Sorting

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

See also the [pagination documentation](../pagination-filters-sorting/pagination.md#disabling-the-pagination).

### Globally

The pagination can be disabled for all GraphQL resources using this configuration:

```yaml
# api/config/packages/api_platform.yaml
api_platform:
    graphql:
        collection:
            pagination:
                enabled: false
```

### For a Specific Resource

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

### For a Specific Resource Collection Operation

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

## Filters & Sorting

Filters are supported out-of-the-box. Follow the [filters](../pagination-filters-sorting/index.md) documentation and your filters will be available as arguments of queries.

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
*          "item_query",
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

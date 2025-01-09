# Pagination

<p align="center" class="symfonycasts"><a href="https://symfonycasts.com/screencast/api-platform/pagination?cid=apip"><img src="../distribution/images/symfonycasts-player.png" alt="Pagination screencast"><br>Watch the Pagination screencast</a></p>

API Platform Core has native support for paged collections. Pagination is enabled by default for all collections. Each collection
contains 30 items per page.
The activation of the pagination and the number of elements per page can be configured from:

* the server-side (globally or per resource)
* the client-side, via a custom GET parameter (disabled by default)

When issuing a `GET` request on a collection containing more than 1 page (here `/books`), a [Hydra collection](http://www.hydra-cg.com/spec/latest/core/#collections)
is returned. It's a valid JSON(-LD) document containing items of the requested page and metadata.

```json
{
  "@context": "/contexts/Book",
  "@id": "/books",
  "@type": "hydra:Collection",
  "hydra:member": [
    {
      "@id": "/books/1",
      "@type": "https://schema.org/Book",
      "name": "My awesome book"
    },
    {
        "_": "Other items in the collection..."
    },
  ],
  "hydra:totalItems": 50,
  "hydra:view": {
    "@id": "/books?page=1",
    "@type": "hydra:PartialCollectionView",
    "hydra:first": "/books?page=1",
    "hydra:last": "/books?page=2",
    "hydra:next": "/books?page=2"
  }
}
```

Hypermedia links to the first, the last, previous and the next page in the collection are displayed as well as the number
of total items in the collection.

The name of the page parameter can be changed with the following configuration:

```yaml
# config/packages/api_platform.yaml
api_platform:
    collection:
        pagination:
            page_parameter_name: _page
```

## Disabling the Pagination

Paginating collections is generally accepted as a good practice. It allows browsing large collections without too much
overhead as well as preventing [DOS attacks](https://en.wikipedia.org/wiki/Denial-of-service_attack).
However, for small collections, it can be convenient to fully disable the pagination.

### Disabling the Pagination Globally

The pagination can be disabled for all resources using this configuration:

```yaml
# config/packages/api_platform.yaml
api_platform:
    defaults:
        pagination_enabled: false
```

### Disabling the Pagination For a Specific Resource

It can also be disabled for a specific resource:

```php
<?php
// api/src/Entity/Book.php

use ApiPlatform\Core\Annotation\ApiResource;

#[ApiResource(attributes: ["pagination_enabled" => false])]
class Book
{
    // ...
}
```

### Disabling the Pagination Client-side

#### Disabling the Pagination Client-side Globally

You can configure API Platform Core to let the client enable or disable the pagination. To activate this feature globally,
use the following configuration:

```yaml
# config/packages/api_platform.yaml
api_platform:
    defaults:
        pagination_client_enabled: true
    collection:
        pagination:
            enabled_parameter_name: pagination # optional
```

The pagination can now be enabled or disabled by adding a query parameter named `pagination`:

* `GET /books?pagination=false`: disabled
* `GET /books?pagination=true`: enabled

Any value accepted by the [`FILTER_VALIDATE_BOOLEAN`](https://www.php.net/manual/en/filter.filters.validate.php) filter can be
used as the value.

#### Disabling the Pagination Client-side For a Specific Resource

The client ability to disable the pagination can also be set in the resource configuration:

```php
<?php
// api/src/Entity/Book.php

use ApiPlatform\Core\Annotation\ApiResource;

#[ApiResource(attributes: ["pagination_client_enabled" => true])]
class Book
{
    // ...
}
```

## Changing the Number of Items per Page

In the same manner, the number of items per page is configurable and can be set client-side.

### Changing the Number of Items per Page Globally

The number of items per page can be configured for all resources:

```yaml
# config/packages/api_platform.yaml
api_platform:
    defaults:
        pagination_items_per_page: 30 # Default value
```

### Changing the Number of Items per Page For a Specific Resource

```php
<?php
// api/src/Entity/Book.php

use ApiPlatform\Core\Annotation\ApiResource;

#[ApiResource(attributes: ["pagination_items_per_page" => 30])]
class Book
{
    // ...
}
```

### Changing the Number of Items per Page Client-side

#### Changing the Number of Items per Page Client-side Globally

```yaml
# config/packages/api_platform.yaml
api_platform:
    defaults:
        pagination_client_items_per_page: true
    collection:
        pagination:
            items_per_page_parameter_name: itemsPerPage # Default value
```

The number of items per page can now be changed adding a query parameter named `itemsPerPage`: `GET /books?itemsPerPage=20`.

#### Changing the Number of Items per Page Client-side For a Specific Resource

Changing the number of items per page can be enabled (or disabled) for a specific resource:

```php
<?php
// api/src/Entity/Book.php

use ApiPlatform\Core\Annotation\ApiResource;

#[ApiResource(attributes: ["pagination_client_items_per_page" => true])]
class Book
{
    // ...
}
```

## Changing Maximum Items Per Page

### Changing Maximum Items Per Page Globally

The number of maximum items per page can be configured for all resources:

```yaml
# config/packages/api_platform.yaml
api_platform:
    defaults:
        pagination_maximum_items_per_page: 50
```

### Changing Maximum Items Per Page For a Specific Resource

```php
<?php
// api/src/Entity/Book.php

use ApiPlatform\Core\Annotation\ApiResource;

#[ApiResource(attributes: ["pagination_maximum_items_per_page" => 50])]
class Book
{
    // ...
}
```

### Changing Maximum Items Per Page For a Specific Resource Collection Operation

```php
<?php
// api/src/Entity/Book.php

use ApiPlatform\Core\Annotation\ApiResource;

#[ApiResource(collectionOperations: [
  "get" => ["pagination_maximum_items_per_page" => 50]
])]
)
class Book
{
    // ...
}
```

## Partial Pagination

When using the default pagination, a `COUNT` query will be issued against the current requested collection. This may have a
performance impact on really big collections. The downside is that the information about the last page is lost (ie: `hydra:last`).

### Partial Pagination Globally

The partial pagination retrieval can be configured for all resources:

```yaml
# config/packages/api_platform.yaml

api_platform:
    defaults:
        pagination_partial: true # Disabled by default
```

### Partial Pagination For a Specific Resource

```php
<?php

// api/src/Entity/Book.php

use ApiPlatform\Core\Annotation\ApiResource;

#[ApiResource(attributes: ["pagination_partial" => true])]
class Book
{
    // ...
}
```

### Partial Pagination Client-side

#### Partial Pagination Client-side Globally

```yaml
# config/packages/api_platform.yaml

api_platform:
    defaults:
        pagination_client_partial: true # Disabled by default
    collection:
        pagination:
            partial_parameter_name: 'partial' # Default value
```

The partial pagination retrieval can now be changed by toggling a query parameter named `partial`: `GET /books?partial=true`.

#### Partial Pagination Client-side For a Specific Resource

```php
<?php

// api/src/Entity/Book.php

use ApiPlatform\Core\Annotation\ApiResource;

#[ApiResource(attributes: ["pagination_client_partial" => true])]
class Book
{
    // ...
}
```

## Cursor-Based Pagination

To configure your resource to use the cursor-based pagination, select your unique sorted field as well as the direction you’ll like the pagination to go via filters and enable the `pagination_via_cursor` option.
Note that for now you have to declare a `RangeFilter` and an `OrderFilter` on the property used for the cursor-based pagination.

The following configuration also works on a specific operation:

```php
<?php

// api/src/Entity/Book.php

use ApiPlatform\Core\Annotation\ApiFilter;
use ApiPlatform\Core\Annotation\ApiResource;
use ApiPlatform\Core\Bridge\Doctrine\MongoDbOdm\Filter\OrderFilter;
use ApiPlatform\Core\Bridge\Doctrine\MongoDbOdm\Filter\RangeFilter;

#[ApiResource(attributes: [
  "pagination_partial" => true,
  "pagination_via_cursor" => [
    ["field" => "id", "direction" => "DESC"],
  ],
)]
#[ApiFilter(RangeFilter::class, properties: ["id"])]
#[ApiFilter(OrderFilter::class, properties: ["id" => "DESC"])]
class Book
{
    // ...
}
```

To know more about cursor-based pagination take a look at [this blog post on medium (draft)](https://medium.com/@sroze/74fd1d324723).

## Controlling The Behavior of The Doctrine ORM Paginator

The [PaginationExtension](https://github.com/api-platform/core/blob/2.6/src/Bridge/Doctrine/Orm/Extension/PaginationExtension.php) of API Platform performs some checks on the `QueryBuilder` to guess, in most common cases, the correct values to use when configuring the Doctrine ORM Paginator:

* `$fetchJoinCollection` argument: Whether there is a join to a collection-valued association. When set to `true`, the Doctrine ORM Paginator will perform an additional query, in order to get the correct number of results.

    You can configure this using the `pagination_fetch_join_collection` attribute on a resource or on a per-operation basis:

    ```php
    <?php
    // api/src/Entity/Book.php

    use ApiPlatform\Core\Annotation\ApiResource;

    #[ApiResource(
      attributes: ["pagination_fetch_join_collection" => false],
      collectionOperations: [
        "get",
        "get_custom" => [
          // ...
          "pagination_fetch_join_collection" => true,
        ],
      ],
    )]
    class Book
    {
        // ...
    }
    ```

* `setUseOutputWalkers` setter: Whether to use output walkers. When set to `true`, the Doctrine ORM Paginator will use output walkers, which are compulsory for some types of queries.

    You can configure this using the `pagination_use_output_walkers` attribute on a resource or on a per-operation basis:

    ```php
    <?php
    // api/src/Entity/Book.php

    use ApiPlatform\Core\Annotation\ApiResource;

    #[ApiResource(
      attributes: ["pagination_use_output_walkers" => false],
      collectionOperations: [
        "get",
        "get_custom" => [
          // ...
          "pagination_use_output_walkers" => true,
        ],
      ],
    )]
    class Book
    {
        // ...
    }
    ```

For more information, please see the [Pagination](https://www.doctrine-project.org/projects/doctrine-orm/en/current/tutorials/pagination.html) entry in the Doctrine ORM documentation.

## Custom Controller Action

In case you're using a custom controller action, make sure you return the `Paginator` object to get the full hydra response with `hydra:view` (which contains information about first, last, next and previous page). The following examples show how to handle it within a repository method.
The controller needs to pass through the page number. You will need to use the Doctrine Paginator and pass it to the API Platform Paginator.

First example:

```php
<?php

// api/src/Repository/BookRepository.php

namespace App\Repository;

use App\Entity\Book;
use Doctrine\Bundle\DoctrineBundle\Repository\ServiceEntityRepository;
use Doctrine\Persistence\ManagerRegistry;
use Symfony\Component\Security\Core\Authentication\Token\Storage\TokenStorageInterface;
use Doctrine\ORM\Tools\Pagination\Paginator as DoctrinePaginator;
use ApiPlatform\Core\Bridge\Doctrine\Orm\Paginator;
use Doctrine\Common\Collections\Criteria;

class BookRepository extends ServiceEntityRepository
{
    const ITEMS_PER_PAGE = 20;

    private $tokenStorage;

    public function __construct(
        ManagerRegistry $registry,
        TokenStorageInterface $tokenStorage
    ) {
        parent::__construct($registry, Book::class);

        $this->tokenStorage = $tokenStorage;
    }

    public function getBooksByFavoriteAuthor(int $page = 1): Paginator
    {
        $firstResult = ($page -1) * self::ITEMS_PER_PAGE;

        $user = $this->tokenStorage->getToken()->getUser();
        $queryBuilder = $this->createQueryBuilder();
        $queryBuilder->select('b')
            ->from(Book::class, 'b')
            ->where('b.author = :author')
            ->setParameter('author', $user->getFavoriteAuthor()->getId())
            ->andWhere('b.publicatedOn IS NOT NULL');

        $criteria = Criteria::create()
            ->setFirstResult($firstResult)
            ->setMaxResults(self::ITEMS_PER_PAGE);
        $queryBuilder->addCriteria($criteria);

        $doctrinePaginator = new DoctrinePaginator($queryBuilder);
        $paginator = new Paginator($doctrinePaginator);

        return $paginator;
    }
}
```

The Controller would look like this:

```php
<?php

// api/src/Controller/Book/GetBooksByFavoriteAuthorAction.php

namespace App\Controller\Book;

use ApiPlatform\Core\Bridge\Doctrine\Orm\Paginator;
use App\Repository\BookRepository;
use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
use Symfony\Component\HttpKernel\Attribute\AsController;
use Symfony\Component\HttpFoundation\Request;

#[AsController]
class GetBooksByFavoriteAuthorAction extends AbstractController
{
    public function __invoke(Request $request, BookRepository $bookRepository): Paginator
    {
        $page = (int) $request->query->get('page', 1);

        return $bookRepository->getBooksByFavoriteAuthor($page);
    }
}
```

The service needs to use the proper repository method.
You can also use the Query object inside the repository method and pass it to the Paginator instead of passing the QueryBuilder and using Criteria. Second Example:

```php
<?php
// api/src/Repository/BookRepository.php

namespace App\Repository;

// use...

class BookRepository extends ServiceEntityRepository
{
    // constant, variables and constructor...

    public function getBooksByFavoriteAuthor(int $page = 1): Paginator
    {
        $firstResult = ($page -1) * self::ITEMS_PER_PAGE;

        $user = $this->tokenStorage->getToken()->getUser();
        $queryBuilder = $this->createQueryBuilder();
        $queryBuilder->select('b')
            ->from(Book::class, 'b')
            ->where('b.author = :author')
            ->setParameter('author', $user->getFavoriteAuthor()->getId())
            ->andWhere('b.publicatedOn IS NOT NULL');

        $query = $queryBuilder->getQuery()
            ->setFirstResult($firstResult)
            ->setMaxResults(self::ITEMS_PER_PAGE);

        $doctrinePaginator = new DoctrinePaginator($query);
        $paginator = new Paginator($doctrinePaginator);

        return $paginator;
    }
}
```

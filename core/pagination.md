# Pagination

API Platform Core has native support for paged collections. Pagination is enabled by default for all collections. Each collections
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
      "@type": "http://schema.org/Book",
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
# app/config/config.yml

api_platform:
    collection:
        pagination:
            page_parameter_name: _page
```

## Disabling the Pagination

Paginating collection is generally accepted as a good practice. It also allows browsing large collections without to much
overhead as well as preventing [DOS attacks](https://en.wikipedia.org/wiki/Denial-of-service_attack).
It allows to browse large collections and prevent. However, for small collections, it can be convenient to fully disable
the pagination.

### Globally

The pagination can be disabled for all resources using this configuration:

```yaml
# app/config/config.yml

api_platform:
    collection:
        pagination:
            enabled: false
```

### For a Specific Resource

It can also be disabled for specific resource:

```php
<?php

// src/AppBundle/Entity/Book.php

use ApiPlatform\Core\Annotation\ApiResource;

/**
 * @ApiResource(attributes={"pagination_enabled"=false})
 */
class Book
{
    // ...
}
```

### Client-side

#### Globally

You can configure API Platform Core to let the client enable or disable the pagination. To activate this feature globally,
use the following configuration:

```yaml
# app/config/config.yml

api_platform:
    collection:
        pagination:
            client_enabled: true
            enabled_parameter_name: pagination # optional
```

The pagination can now be enabled or disabled by adding a query parameter named `pagination`:

* `GET /books?pagination=false`: disabled
* `GET /books?pagination=true`: enabled

Any value accepted by the [`FILTER_VALIDATE_BOOLEAN`](http://php.net/manual/en/filter.filters.validate.php) filter can be
used as the value.

#### For a specific resource

The client ability to disable the pagination can also be set in the resource configuration:

```php
<?php

// src/AppBundle/Entity/Book.php

use ApiPlatform\Core\Annotation\ApiResource;

/**
 * @ApiResource(attributes={"pagination_client_enabled"=true})
 */
class Book
{
    // ...
}
```

## Changing the Number of Items per Page

In the same manner, the number of items per page is configurable and can be set client-side.

### Globally

The number of items per page can be configured for all resources:

```yaml
# app/config/config.yml

api_platform:
    collection:
        pagination:
            items_per_page: 30 # Default value
```

### For a Specific Resource

```php
<?php

// src/AppBundle/Entity/Book.php

use ApiPlatform\Core\Annotation\ApiResource;

/**
 * @ApiResource(attributes={"pagination_items_per_page"=30})
 */
class Book
{
    // ...
}
```

### Client-side

#### Globally

```yaml
# app/config/config.yml

api_platform:
    collection:
        pagination:
            client_items_per_page: true # Disabled by default
            items_per_page_parameter_name: itemsPerPage # Default value
```

The number of items per page can now be changed adding a query parameter named `itemsPerPage`: `GET /books?itemsPerPage=20`

#### For a Specific Resource

Changing the number of items per page can be enabled (or disabled) for a specific resource:

```php
<?php

// src/AppBundle/Entity/Book.php

use ApiPlatform\Core\Annotation\ApiResource;

/**
 * @ApiResource(attributes={"pagination_client_items_per_page"=true})
 */
class Book
{
    // ...
}
```

Previous chapter: [Validation](validation.md)

Next chapter: [Error Handling](errors.md)

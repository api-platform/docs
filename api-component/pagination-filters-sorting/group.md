## Group Filter

The group filter allows you to filter by serialization groups.

Syntax: `?groups[]=<group>`

You can add as many groups as you need.

## Implementations

This filter is not dependent of your persistence logic and works with any data provider.

## Usage

Enable the filter:

```php
<?php
// api/src/Entity/Book.php

namespace App\Entity;

use ApiPlatform\Core\Annotation\ApiFilter;
use ApiPlatform\Core\Annotation\ApiResource;
use ApiPlatform\Core\Serializer\Filter\GroupFilter;

/**
 * @ApiResource
 * @ApiFilter(GroupFilter::class, arguments={"parameterName": "groups", "overrideDefaultGroups": false, "whitelist": {"allowed_group"}})
 */
class Book
{
    // ...
}
```

Three arguments are available to configure the filter:
- `parameterName` is the query parameter name (default `groups`)
- `overrideDefaultGroups` allows to override the default serialization groups (default `false`)
- `whitelist` groups whitelist to avoid uncontrolled data exposure (default `null` to allow all groups)

Given that the collection endpoint is `/books`, you can filter by serialization groups with the following query: `/books?groups[]=read&groups[]=write`.

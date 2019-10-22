# Property filter

The property filter adds the possibility to select the properties to serialize (sparse fieldsets).

Syntax: `?properties[]=<property>&properties[<relation>][]=<property>`

You can add as many properties as you need.

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
use ApiPlatform\Core\Serializer\Filter\PropertyFilter;

/**
 * @ApiResource
 * @ApiFilter(PropertyFilter::class, arguments={"parameterName": "properties", "overrideDefaultProperties": false, "whitelist": {"allowed_property"}})
 */
class Book
{
    // ...
}
```

Three arguments are available to configure the filter:
- `parameterName` is the query parameter name (default `properties`)
- `overrideDefaultProperties` allows to override the default serialization properties (default `false`)
- `whitelist` properties whitelist to avoid uncontrolled data exposure (default `null` to allow all properties)

Given that the collection endpoint is `/books`, you can filter the serialization properties with the following query: `/books?properties[]=title&properties[]=author`.
If you want to include some properties of the nested "author" document, use: `/books?properties[]=title&properties[author][]=name`.

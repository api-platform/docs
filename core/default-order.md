# Override default order

API Platform Core provides an easy way to override default order in your collection.

By default, it will order by resource identifier(s) using ASC direction. If you want to customize this order, you must add an `order` attribute on your ApiResource annotation:

```php
<?php

// src/AppBundle/Entity/Book.php

namespace AppBundle\Entity;

use ApiPlatform\Core\Annotation\ApiProperty;
use ApiPlatform\Core\Annotation\ApiResource;

/**
 * @ApiResource(attributes={"order"={"foo", "ASC"}})
 */
class Book
{
    // ...

    /**
     * ...
     * @ApiProperty()
     */
    public $foo;
}
```

This `order` attribute is used as an array with 2 entries: the first one defines the order field, the second one defined the direction.
If you only specify the first one, `ASC` direction will be used as default.

Previous chapter: [Configuration](configuration.md)

Next chapter: [Operations](operations.md)

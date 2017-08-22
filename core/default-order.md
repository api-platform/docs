# Override Default Order

API Platform Core provides an easy way to override default order in your collection.

By default, it will order by resource identifier(s) using ASC direction. If you want to customize this order, you must add an `order` attribute on your ApiResource annotation:

```php
<?php

// src/AppBundle/Entity/Book.php

namespace AppBundle\Entity;

use ApiPlatform\Core\Annotation\ApiResource;

/**
 * @ApiResource(attributes={"order"={"foo": "ASC"}})
 */
class Book
{
    // ...

    /**
     * ...
     */
    public $foo;
}
```

This `order` attribute is used as an array: the key defines the order field, the values defines the direction.
If you only specify the key, `ASC` direction will be used as default. For example, to order by `foo` & `bar`:

```php
<?php

// src/AppBundle/Entity/Book.php

namespace AppBundle\Entity;

use ApiPlatform\Core\Annotation\ApiResource;

/**
 * @ApiResource(attributes={"order"={"foo", "bar"}})
 */
class Book
{
    // ...

    /**
     * ...
     */
    public $foo;

    /**
     * ...
     */
    public $bar;
}
```

It's also possible to configure the default filter on an association property:

```php
<?php

// src/AppBundle/Entity/Book.php

namespace AppBundle\Entity;

use ApiPlatform\Core\Annotation\ApiResource;

/**
 * @ApiResource(attributes={"order"={"author.username"}})
 */
class Book
{
    // ...

    /**
     * @var User
     */
    public $author;
}
```

Previous chapter: [Configuration](configuration.md)

Next chapter: [Operations](operations.md)

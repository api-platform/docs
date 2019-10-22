# Dispatching Private Updates (Authorized Mode)

Mercure allows to dispatch [private updates, that will be received only by authorized clients](https://github.com/dunglas/mercure/blob/master/spec/mercure.md#authorization).
To receive this kind of updates, the client must hold a JWT containing at least one *target* marking the update.

Then, hint API Platform to dispatch the updates only to the selected targets:

```php
<?php
// api/src/Entity/Book.php

namespace App\Entity;

use ApiPlatform\Core\Annotation\ApiResource;

/**
 * @ApiResource(mercure={"a-group", "another-group"})
 */
class Book
{
    // ...
}
```

It's also possible to execute an *expression* (using the [Symfony Expression Language component](https://symfony.com/doc/current/components/expression_language.html)), to generate a dynamic list of targets:

```php
<?php
// api/src/Entity/Book.php

namespace App\Entity;

use ApiPlatform\Core\Annotation\ApiResource;

/**
 * @ApiResource(mercure="object.owners")
 */
class Book
{
    public $owners = [];

   // ...
}
```

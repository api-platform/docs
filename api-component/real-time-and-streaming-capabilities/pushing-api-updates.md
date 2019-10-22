# Pushing the API Updates

Use the `mercure` attribute to hint API Platform that it must dispatch the updates regarding the given resources to the Mercure hub:

```php
<?php
// api/src/Entity/Book.php

namespace App\Entity;

use ApiPlatform\Core\Annotation\ApiResource;

/**
 * @ApiResource(mercure=true)
 */
class Book
{
    // ...
}
```

Then, every time an object of this type is created, updated or deleted, the new version is sent to all connected clients through the Mercure hub.
If the resource has been deleted, only the (now deleted) IRI of the resource is sent to the clients.

In addition, API Platform automatically adds a `Link` HTTP header to all responses related to this resource class.
This header allows smart clients to automatically discover the Mercure hub.

![Mercure subscriptions](images/mercure-discovery.png)

Clients generated using [the API Platform Client Generator](../../client-generator/index.md) will use this capability to automatically subscribe to Mercure updates when available:

![Screencast](../../client-generator/images/client-generator-demo.gif)

[Learn how to use the discovery capabilities of Mercure in your own clients](https://github.com/dunglas/mercure#examples).

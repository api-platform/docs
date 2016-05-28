# Controllers

The library provides default action classes implementing CRUD operations: `ApiPlatform\Core\Action`.
They provide convenient methods to retrieve the `Resource` associated with the current request and to serialize entities
using the normalizers provided by API Platform.

## Using a custom controller

When [the event system](the-event-system.md) is not enough, it is possible to use custom controllers.

Your custom controller should extend the `Controller` provided by Symfony.

Example of a custom controller:

```php
<?php

// src/AppBundle/Controller/CustomController.php

namespace AppBundle\Controller;

use ApiPlatform\Core\JsonLd\Response;
use Symfony\Bundle\FrameworkBundle\Controller\Controller;

class CustomController extends Controller
{
    /**
     * @return \ApiPlatform\Core\JsonLd\Response
     */
    public function customAction($id)
    {
        return new Response(sprintf('This is a custom action for %d.', $id));
    }
}
```

Custom controllers are often used with [custom operations](operations.md). If you don't create a custom operation
for your custom controller, in order for it to appear in the documentation, you need to register that controller in 
the Symfony routing system yourself.

Note that you shouldn't use the `@Route` annotations, as this will cause bugs. The bundle takes care of registering routes in Symfony, so you don't need to do it yourself.

Previous chapter: [Resources](resources.md)<br>
Next chapter: [Using external (JSON-LD) vocabularies](external-vocabularies.md)

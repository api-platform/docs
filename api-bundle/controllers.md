# Controllers

The bundle provides custom Json-LD response using the Symfony `Response` and `Controller`
WIP : @todo clarify that

## Using a custom controller

When [the event system](the-event-system.md) is not enough, it's possible to use custom controllers.

Your custom controller should extend the `Controller` provided by Symfony.

Example of custom controller:

```php
<?php

// src/AppBundle/Controller/CustomController.php

namespace AppBundle\Controller;

use ApiPlatform\Core\JsonLd\Response;
use Symfony\Bundle\FrameworkBundle\Controller\Controller;

/**
 * Custom Controller.
 *
 * @author Kévin Dunglas <dunglas@gmail.com>
 */
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

Note that you shouldn't use `@Route` annotations, as this will cause bugs. The bundle auto-registers routes within Symfony2, so you don't need to use `@Route` annotations.

Previous chapter: [Resources](resources.md)<br>
Next chapter: [Using external (JSON-LD) vocabularies](external-vocabularies.md)

# Controllers

The bundle provides a default controller class implementing CRUD operations: `Dunglas\ApiBundle\Controller\ResourceController`.
Basically, this controller class extends the default controller class of Symfony's FrameworkBundle. It also provides convenient methods to retrieve the `Resource` class associated with the current request
and to serialize entities using the normalizers provided by the bundle.

## Using a custom controller

When [the event system](the-event-system.md) is not enough, it is possible to use custom controllers.

Your custom controller should extend the `ResourceController` provided by this bundle.

Example of a custom controller:

```php
<?php

// src/AppBundle/Controller/CustomController.php

namespace AppBundle\Controller;

use Dunglas\ApiBundle\Controller\ResourceController;
use Symfony\Component\HttpFoundation\Request;

class CustomController extends ResourceController
{
    // Customize the AppBundle:Custom:custom action
    public function getAction(Request $request, $id)
    {
        $this->get('logger')->info('This is my custom controller.');
        
        return parent::getAction($request, $id);
    }
}
```

Custom controllers are often used with [custom operations](operations.md). If you don't create a custom operation
for your custom controller, in order for it to appear in the documentation, you need to register that controller in 
the Symfony routing system yourself.

Note that you shouldn't use the `@Route` annotations, as this will cause bugs. The bundle takes care of registering routes in Symfony, so you don't need to do it yourself.

Previous chapter: [Resources](resources.md)<br>
Next chapter: [Using external (JSON-LD) vocabularies](external-vocabularies.md)

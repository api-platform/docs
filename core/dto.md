# Handling Data Transfer Objects (DTOs)

## How to Use a DTO for Writing

Sometimes it's easier to use a DTO than an Entity when performing simple
operation. For example, the application should be able to send an email when
someone has lost its password.

So let's create a basic DTO for this request:

```php
<?php
// api/src/Api/Dto/ForgotPasswordRequest.php

namespace App\Api\Dto;

use ApiPlatform\Core\Annotation\ApiResource;
use Symfony\Component\Validator\Constraints as Assert;

/**
 * @ApiResource(
 *      collectionOperations={
 *          "post"={
 *              "path"="/users/forgot-password-request",
 *          },
 *      },
 *      itemOperations={},
 * )
 */
final class ForgotPasswordRequest
{
    /**
     * @Assert\NotBlank
     * @Assert\Email
     */
    public $email;
}
```

In this case, we disable all operations except `POST`.

Then, thanks to [the event system](events.md), it's possible to intercept the
`POST` request and to handle it.

First, we should define a custom loader paths and create an event subscriber:

* define a custom loader paths for `Api/Dto`:

```yaml
api_platform:
    mapping:
        paths: ['%kernel.project_dir%/src/Api/Dto']
```

* create an event subscriber:

```php
<?php
// api/src/Api/EventSubscriber/UserSubscriber.php

namespace App\Api\EventSubscriber;

use ApiPlatform\Core\EventListener\EventPriorities;
use App\Entity\User;
use App\Manager\UserManager;
use Symfony\Component\EventDispatcher\EventSubscriberInterface;
use Symfony\Component\HttpFoundation\JsonResponse;
use Symfony\Component\HttpKernel\Event\GetResponseForControllerResultEvent;
use Symfony\Component\HttpKernel\KernelEvents;

final class UserSubscriber implements EventSubscriberInterface
{
    private $userManager;

    public function __construct(UserManager $userManager)
    {
        $this->userManager = $userManager;
    }

    public static function getSubscribedEvents()
    {
        return [
            KernelEvents::VIEW => ['sendPasswordReset', EventPriorities::POST_VALIDATE],
        ];
    }

    public function sendPasswordReset(GetResponseForControllerResultEvent $event)
    {
        $request = $event->getRequest();

        if ('api_forgot_password_requests_post_collection' !== $request->attributes->get('_route')) {
            return;
        }

        $forgotPasswordRequest = $event->getControllerResult();

        $user = $this->userManager->findOneByEmail($forgotPasswordRequest->email);

        // We do nothing if the user does not exist in the database
        if ($user) {
            $this->userManager->requestPasswordReset($user);
        }

        $event->setResponse(new JsonResponse(null, 204));
    }
}
```

Then this class should be registered as a service, then tagged.

If service autowiring and autoconfiguration are enabled (it's the case by
default), you are done!

Otherwise, the following configuration is needed:

```yaml
# api/config/services.yaml
services:
    # ...
    'App\Api\EventSubscriber\UserSubscriber':
        arguments:
            - '@app.manager.user'
        # Uncomment the following line only if you don't use autoconfiguration
        #tags: [ 'kernel.event_subscriber' ]
```

## How to Use a DTO for Reading

Sometimes, you need to retrieve data not related to an entity.
For example, the application can send the
[list of supported locales](https://github.com/symfony/demo/blob/master/config/services.yaml#L6)
and the default locale.

So let's create a basic DTO for this datas:

```php
<?php
// api/src/Dto/LocalesList.php

namespace App\Dto;

final class LocalesList
{
    /**
     * @var array
     */
    public $locales;

    /**
     * @var string
     */
    public $defaultLocale;
}
```

And create a controller to send them:

```php
<?php
// api/src/Controller/LocaleController.php

namespace App\Controller;

use App\DTO\LocalesList;
use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
use Symfony\Component\Routing\Annotation\Route;

class LocaleController extends AbstractController
{
    /**
     * @Route(
     *     path="/api/locales",
     *     name="api_get_locales",
     *     methods={"GET"},
     *     defaults={
     *          "_api_respond"=true,
     *          "_api_normalization_context"={"api_sub_level"=true}
     *     }
     * )
     */
    public function __invoke(): LocalesDTO
    {
        $response = new LocalesList();
        $response->locales = explode('|', $this->getParameter('app_locales'));
        $response->defaultLocale = $this->getParameter('locale');

        return $response;
    }
}
```

As you can see, the controller doesn't return a `Response`, but the data object directly.
Behind the scene, the `SerializeListener` catch the response, and thanks to the `_api_respond`
flag, it serializes the object correctly.

To deal with arrays, we have to set the `api_sub_level` context option to `true`.
It prevents API Platform's normalizers to look for a non-existing class marked as an API resource.

### Adding this Custom DTO reading in Swagger Documentation.

By default, ApiPlatform Swagger UI integration will display documentation only
for ApiResource operations.
In this case, our DTO is not declared as ApiResource, so no documentation will
be displayed.

There is two solutions to achieve that:

#### Use Swagger Decorator

By following the doc about [Override the Swagger Documentation](swagger.md#overriding-the-swagger-documentation)
and adding the ability to retrieve a `_api_swagger_context` in route
parameters, you should be able to display your custom endpoint.

```php
<?php
// src/App/Swagger/ControllerSwaggerDecorator

namespace App\Swagger;

use Symfony\Component\Routing\RouterInterface;
use Symfony\Component\Serializer\Normalizer\NormalizerInterface;

final class ControllerSwaggerDecorator implements NormalizerInterface
{
    private $decorated;

    private $router;

    public function __construct(
        NormalizerInterface $decorated,
        RouterInterface $router
    ) {
        $this->decorated = $decorated;
        $this->router = $router;
    }

    public function normalize($object, $format = null, array $context = [])
    {
        $docs = $this->decorated->normalize($object, $format, $context);
        $mimeTypes = $object->getMimeTypes();
        foreach ($this->router->getRouteCollection()->all() as $routeName => $route) {
            $swaggerContext = $route->getDefault('_api_swagger_context');
            if (!$swaggerContext) {
                // No swagger_context set, continue
                continue;
            }

            $methods = $route->getMethods();
            $uri = $route->getPath();

            foreach ($methods as $method) {
                // Add available mimesTypes
                $swaggerContext['produces'] ?? $swaggerContext['produces'] = $mimeTypes;

                $docs['paths'][$uri][\strtolower($method)] = $swaggerContext;
            }
        }

        return $docs;
    }

    public function supportsNormalization($data, $format = null)
    {
        return $this->decorated->supportsNormalization($data, $format);
    }
}
```

Register it as a service:

```yaml
#config/services.yaml
# ...
    'App\Swagger\ControllerSwaggerDecorator':
        decorates: 'api_platform.swagger.normalizer.documentation'
        arguments: [ '@App\Swagger\ControllerSwaggerDecorator.inner']
        autoconfigure: false
```

And finally, complete the Route annotation of your controller like this:

```php
<?php
// api/src/Controller/LocaleController.php

use Nelmio\ApiDocBundle\Annotation\Model;
use Swagger\Annotations as SWG;

//...

    /**
     * @Route(
     *     path="/api/locales",
     *     name="api_get_locales",
     *     methods={"GET"},
     *     defaults={
     *          "_api_respond"=true,
     *          "_api_normalization_context"={"api_sub_level"=true},
     *          "_api_swagger_context"={
     *              "tags"={"Locales"},
     *              "summary"="Retrieve locales availables",
     *              "parameters"={},
     *              "responses"={
     *                  "200"={
     *                      "description"="List of available locales and the default locale",
     *                      "schema"={
     *                          "type"="object",
     *                          "properties"={
     *                              "defaultLocale"={"type"="string"},
     *                          }
     *                      }
     *                  }
     *              }
     *          }
     *     }
     * )
     */
    public function __invoke()
```

#### Use [NelmioApiDoc](nelmio-api-doc.md)

With NelmioApiDoc, you should be able to add annotations to your controllers :

```php
<?php
// api/src/Controller/LocaleController.php

use Nelmio\ApiDocBundle\Annotation\Model;
use Swagger\Annotations as SWG;

//...

    /**
     * @Route(
     *     path="/api/locales",
     *     name="api_get_locales",
     *     methods={"GET"},
     *     defaults={
     *          "_api_respond"=true,
     *          "_api_normalization_context"={"api_sub_level"=true}
     *     }
     * )
     * @SWG\Tag(name="Locales")
     * @SWG\Response(
     *     response=200,
     *     description="List of available locales and the default locale",
     *     @SWG\Schema(ref=@Model(type=LocalesList::class)),
     * )
     */
    public function __invoke()
```

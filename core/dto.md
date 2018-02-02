# Handling Data Transfer Objects (DTOs)

## How to use a DTO for Writing

Sometimes it's easier to use a DTO than an Entity when performing simple
operation. For example, the application should be able to send an email when
someone has lost its password.

So let's create a basic DTO for this request:

```php
// api/src/Api/Dto/ForgotPasswordRequest.php

namespace App\Api\Dto;

use ApiPlatform\Core\Annotation\ApiResource;
use Symfony\Component\Validator\Constraints as Assert;

/**
 * @ApiResource(
 *      collectionOperations={
 *          "post"={
 *              "method"="POST",
 *              "path"="/users/forgot-password-request",
 *          },
 *      },
 *      itemOperations={},
 * )
 */
final class ForgotPasswordRequest
{
    /**
     * @Assert\NotBlank()
     * @Assert\Email()
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
# api/config/services.yml
services:

    # ...

    'App\Api\EventSubscriber\UserSubscriber':
        arguments:
            - '@app.manager.user'
        tags: [ 'kernel.event_subscriber' ]
```

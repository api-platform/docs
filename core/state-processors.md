# State processors

To mutate the application states during `POST`, `PUT`, `PATCH` or `DELETE` [operations](operations.md), API Platform uses
classes called **state processors**. State processors receive an instance of the class marked as an API resource (usually using
the `#[ApiResource]` attribute). This instance contains data submitted by the client during [the deserialization
process](serialization.md).

A state processor using [Doctrine ORM](https://www.doctrine-project.org/projects/orm.html) is included with the library and
is enabled by default. It is able to persist and delete objects that are also mapped as [Doctrine entities](https://www.doctrine-project.org/projects/doctrine-orm/en/current/reference/basic-mapping.html).
A [Doctrine MongoDB ODM](https://www.doctrine-project.org/projects/mongodb-odm.html) state processor is also included and can be enabled by following the [MongoDB documentation](mongodb.md).

However, you may want to:

* store data to other persistence layers (Elasticsearch, external web services...)
* not publicly expose the internal model mapped with the database through the API
* use a separate model for [read operations](data-providers.md) and for updates by implementing patterns such as [CQRS](https://martinfowler.com/bliki/CQRS.html)

Custom state processors can be used to do so. A project can include as many state processors as needed. The first able to
process the data for a given resource will be used.

## Creating a Custom State Processor

If the [Symfony MakerBundle](https://symfony.com/doc/current/bundles/SymfonyMakerBundle) is installed in your project, you can use the following command to generate a custom state processor easily:

```console
bin/console make:state-processor
```

To create a state processor, you have to implement the [`ProcessorInterface`](https://github.com/api-platform/core/blob/main/src/State/ProcessorInterface.php).
This interface defines only 3 methods:

* `process`: to create, delete, update, or process the given data in any ways
* `supports`: to check whether the given data is supported by this state processor

Here is an implementation example:

```php
<?php

namespace App\State;

use App\Entity\BlogPost;
use ApiPlatform\State\ProcessorInterface;

class BlogPostProcessor implements ProcessorInterface
{
    /**
     * {@inheritDoc}
     */
    public function process($data, array $identifiers = [], ?string $operationName = null, array $context = [])
    {
        // call your persistence layer to save $data
        return $data;
    }

    /**
     * {@inheritDoc}
     */
    public function supports($data, array $identifiers = [], ?string $operationName = null, array $context = []): bool
    {
        return $data instanceof BlogPost && '_api_/blog_posts_post' === $operationName;
    }
}
```

You can find the operation name information either with the `debug:router` command (the route name and the operation name are
the same), or by using the `debug:api` command. 
If service autowiring and autoconfiguration are enabled (they are by default), you are done!

Otherwise, if you use a custom dependency injection configuration, you need to register the corresponding service and add the
`api_platform.state_processor` tag. The `priority` attribute can be used to order persisters.

```yaml
# api/config/services.yaml
services:
    # ...
    App\State\BlogPostProcessor: ~
        # Uncomment only if autoconfiguration is disabled
        #tags: [ 'api_platform.state_processor' ]
```

## Decorating the Built-In Data Persisters

If you want to execute custom business logic before or after persistence, this can be achieved by [decorating](https://symfony.com/doc/current/service_container/service_decoration.html) the built-in data persisters.

The next example uses [Symfony Mailer](https://symfony.com/doc/current/mailer.html). Read its documentation if you want to use it.

Here is an implementation example which sends new users a welcome email after a REST `POST` or GraphQL `create` operation, in a project using the native Doctrine ORM data persister:

```php
<?php

namespace App\State;

use ApiPlatform\State\ProcessorInterface;
use App\Entity\User;
use Symfony\Component\Mailer\MailerInterface;

final class UserProcessor implements ProcessorInterface
{
    private $decorated;
    private $mailer;

    public function __construct(ProcessorInterface $decorated, MailerInterface $mailer)
    {
        $this->decorated = $decorated;
        $this->mailer = $mailer;
    }

    public function process($data, array $identifiers = [], ?string $operationName = null, array $context = [])
    {
        $result = $this->decorated->persist($data, $context);

        if ($data instanceof User && '_api_/blog_posts_post' === $operationName) {
            $this->sendWelcomeEmail($data);
        }

        return $result;
    }

    public function supports($data, array $identifiers = [], ?string $operationName = null, array $context = []): bool
    {
        return $this->decorated->supports($data, $identifeirs, $operationName, $context);
    }

    private function sendWelcomeEmail(User $user)
    {
        // Your welcome email logic...
        // $this->mailer->send(...);
    }
}
```

Even with service autowiring and autoconfiguration enabled, you must still configure the decoration:

```yaml
# api/config/services.yaml
services:
    # ...
    App\State\UserProcessor:
        bind:
            $decorated: '@api_platform.doctrine.orm.state_processor'
        # Uncomment only if autoconfiguration is disabled
        #arguments: ['@App\State\UserProcessor.inner']
        #tags: [ 'api_platform.state_processor' ]
```

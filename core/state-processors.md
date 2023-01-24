# State Processors

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
* use a separate model for [read operations](state-providers.md) and for updates by implementing patterns such as [CQRS](https://martinfowler.com/bliki/CQRS.html)

Custom state processors can be used to do so. A project can include as many state processors as needed. The first able to
process the data for a given resource will be used.

## Creating a Custom State Processor

If the [Symfony MakerBundle](https://symfony.com/doc/current/bundles/SymfonyMakerBundle) is installed in your project, you can use the following command to generate a custom state processor easily:

```console
bin/console make:state-processor
```

To create a state processor, you have to implement the [`ProcessorInterface`](https://github.com/api-platform/core/blob/main/src/State/ProcessorInterface.php).
This interface defines a method `process`: to create, delete, update, or alter the given data in any ways.

Here is an implementation example:

```php
<?php

namespace App\State;

use App\Entity\BlogPost;
use ApiPlatform\Metadata\Operation;
use ApiPlatform\State\ProcessorInterface;

class BlogPostProcessor implements ProcessorInterface
{
    /**
     * {@inheritDoc}
     */
    public function process($data, Operation $operation, array $uriVariables = [], array $context = [])
    {
        // call your persistence layer to save $data
        return $data;
    }
}
```

We then configure our operation to use this processor:

```php
<?php

namespace App\Entity;

use ApiPlatform\Metadata\Post;
use App\State\BlogPostProcessor;

#[Post(processor: BlogPostProcessor::class)]
class BlogPost {}
```

If service autowiring and autoconfiguration are enabled (they are by default), you are done!

Otherwise, if you use a custom dependency injection configuration, you need to register the corresponding service and add the
`api_platform.state_processor` tag.

```yaml
# api/config/services.yaml
services:
    # ...
    App\State\BlogPostProcessor: ~
        # Uncomment only if autoconfiguration is disabled
        #tags: [ 'api_platform.state_processor' ]
```

## Hooking into the Built-In State Processors

If you want to execute custom business logic before or after persistence, this can be achieved by [decorating](https://symfony.com/doc/current/service_container/service_decoration.html) the built-in state processors or using [composition](https://en.wikipedia.org/wiki/Object_composition).

The next example uses [Symfony Mailer](https://symfony.com/doc/current/mailer.html). Read its documentation if you want to use it.

Here is an implementation example which sends new users a welcome email after a REST `POST` or GraphQL `create` operation, in a project using the native Doctrine ORM state processor:

```php
<?php

namespace App\State;

use ApiPlatform\Metadata\DeleteOperationInterface;
use ApiPlatform\Metadata\Operation;
use ApiPlatform\State\ProcessorInterface;
use App\Entity\User;
use Symfony\Component\Mailer\MailerInterface;

final class UserProcessor implements ProcessorInterface
{
    public function __construct(private ProcessorInterface $persistProcessor, private ProcessorInterface $removeProcessor, MailerInterface $mailer)
    {
    }

    public function process($data, Operation $operation, array $uriVariables = [], array $context = [])
    {
        if ($operation instanceof DeleteOperationInterface) {
            return $this->removeProcessor->process($data, $operation, $uriVariables, $context);
        }
    
        $result = $this->persistProcessor->process($data, $operation, $uriVariables, $context);
        $this->sendWelcomeEmail($data);
        return $result;
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
            $persistProcessor: '@api_platform.doctrine.orm.state.persist_processor'
            $removeProcessor: '@api_platform.doctrine.orm.state.remove_processor'
        # Uncomment only if autoconfiguration is disabled
        #arguments: ['@App\State\UserProcessor.inner']
        #tags: [ 'api_platform.state_processor' ]
```

And configure that you want to use this processor on the User resource:

```php
<?php

namespace App\Entity;

use ApiPlatform\Metadata\ApiResource;
use App\State\UserProcessor;

#[ApiResource(processor: UserProcessor::class)]
class User {}
```

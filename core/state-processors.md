# State Processors

To mutate the application states during `POST`, `PUT`, `PATCH` or `DELETE` [operations](operations.md), API Platform uses
classes called **state processors**. State processors receive an instance of the class marked as an API resource (usually using
the `#[ApiResource]` attribute). This instance contains data submitted by the client during [the deserialization
process](serialization.md).

With the Symfony variant, a state processor using [Doctrine ORM](https://www.doctrine-project.org/projects/orm.html) is included with the library and
is enabled by default. It is able to persist and delete objects that are also mapped as [Doctrine entities](https://www.doctrine-project.org/projects/doctrine-orm/en/current/reference/basic-mapping.html).
A [Doctrine MongoDB ODM](https://www.doctrine-project.org/projects/mongodb-odm.html) state processor is also included and can be enabled by following the [MongoDB documentation](mongodb.md).

With the Laravel variant, a state processor using [Eloquent ORM](https://laravel.com/docs/eloquent) is included with the library and
is enabled by default. It is able to persist and delete objects that are also mapped as [Related Models](https://laravel.com/docs/eloquent-relationships#inserting-and-updating-related-models).
A [Laravel MongoDB](https://www.mongodb.com/docs/drivers/php/laravel-mongodb/current/) state processor is also included and can be enabled by following the [MongoDB documentation](../core/mongodb.md).

However, you may want to:

* store data to other persistence layers (Elasticsearch, external web services...)
* not publicly expose the internal model mapped with the database through the API
* use a separate model for [read operations](state-providers.md) and for updates by implementing patterns such as [CQRS](https://martinfowler.com/bliki/CQRS.html)

Custom state processors can be used to do so. A project can include as many state processors as needed. The first able to
process the data for a given resource will be used.

## Creating a Custom State Processor

### Custom State Processor with Symfony
If the [Symfony MakerBundle](https://symfony.com/doc/current/bundles/SymfonyMakerBundle) is installed in your project, you can use the following command to generate a custom state processor easily:

```console
bin/console make:state-processor
```

To create a state processor, you have to implement the [`ProcessorInterface`](https://github.com/api-platform/core/blob/main/src/State/ProcessorInterface.php).
This interface defines a method `process`: to create, delete, update, or alter the given data in any ways.

Here is an implementation example:

```php
<?php
// api/src/State/BlogPostProcessor.php

namespace App\State;

use App\Entity\BlogPost;
use ApiPlatform\Metadata\Operation;
use ApiPlatform\State\ProcessorInterface;

/**
 * @implements ProcessorInterface<BlogPost, BlogPost|void>
 */
final class BlogPostProcessor implements ProcessorInterface
{
    /**
     * @return BlogPost|void
     */
    public function process(mixed $data, Operation $operation, array $uriVariables = [], array $context = []): mixed
    {
        // call your persistence layer to save $data
        return $data;
    }
}
```

The `process()` method must return the created or modified object, or nothing (that's why `void` is allowed) for `DELETE` operations.
The `process()` method can also take an object as input, in the `$data` parameter, that isn't of the same type that its output (the returned object). See [the DTO documentation entry](dto.md) for more details.

We then configure our operation to use this processor:

```php
<?php
// api/src/Entity/BlogPost.php

namespace App\Entity;

use ApiPlatform\Metadata\Post;
use App\State\BlogPostProcessor;

#[Post(processor: BlogPostProcessor::class)]
class BlogPost {}
```

### Custom State Processor with Laravel
Using [Laravel Artisan Console](https://laravel.com/docs/artisan), you can generate a custom state processor easily with the following command:
```console
php artisan make:state-processor
```

To create a state processor, you have to implement the [`ProcessorInterface`](https://github.com/api-platform/core/blob/main/src/State/ProcessorInterface.php).
This interface defines a method `process`: to create, delete, update, or alter the given data in any ways.

Here is an implementation example:

```php
<?php
// api/app/State/BlogPostProcessor.php

namespace App\State;

use App\Models\BlogPost;
use ApiPlatform\Metadata\Operation;
use ApiPlatform\State\ProcessorInterface;

/**
 * @implements ProcessorInterface<BlogPost, BlogPost|void>
 */
final class BlogPostProcessor implements ProcessorInterface
{
    /**
     * @return BlogPost|void
     */
    public function process(mixed $data, Operation $operation, array $uriVariables = [], array $context = []): mixed
    {
        // call your persistence layer to save $data
        return $data;
    }
}
```

The `process()` method must return the created or modified object, or nothing (that's why `void` is allowed) for `DELETE` operations.
The `process()` method can also take an object as input, in the `$data` parameter, that isn't of the same type that its output (the returned object). See [the DTO documentation entry](dto.md) for more details.

We then configure our operation to use this processor:

```php
<?php
// api/app/Models/BlogPost.php

namespace App\Models;

use ApiPlatform\Metadata\Post;
use App\State\BlogPostProcessor;

#[Post(processor: BlogPostProcessor::class)]
class BlogPost {}
```

## Hooking into the Built-In State Processors

### Symfony State Provider mechanism
If you want to execute custom business logic before or after persistence, this can be achieved by using [composition](https://en.wikipedia.org/wiki/Object_composition).

Here is an implementation example which uses [Symfony Mailer](https://symfony.com/doc/current/mailer.html) to send new users a welcome email after a REST `POST` or GraphQL `create` operation, in a project using the native Doctrine ORM state processor:

```php
<?php
// api/src/State/UserProcessor.php

namespace App\State;

use ApiPlatform\Metadata\DeleteOperationInterface;
use ApiPlatform\Metadata\Operation;
use ApiPlatform\State\ProcessorInterface;
use App\Entity\User;
use Symfony\Component\DependencyInjection\Attribute\Autowire;
use Symfony\Component\Mailer\MailerInterface;

/**
 * @implements ProcessorInterface<User, User|void>
 */
final class UserProcessor implements ProcessorInterface
{
    public function __construct(
        #[Autowire(service: 'api_platform.doctrine.orm.state.persist_processor')]
        private ProcessorInterface $persistProcessor,
        #[Autowire(service: 'api_platform.doctrine.orm.state.remove_processor')]
        private ProcessorInterface $removeProcessor,
        private MailerInterface $mailer,
    )
    {
    }

    /**
     * @return User|void
     */
    public function process(mixed $data, Operation $operation, array $uriVariables = [], array $context = []): mixed
    {
        if ($operation instanceof DeleteOperationInterface) {
            return $this->removeProcessor->process($data, $operation, $uriVariables, $context);
        }
    
        $result = $this->persistProcessor->process($data, $operation, $uriVariables, $context);
        $this->sendWelcomeEmail($data);

        return $result;
    }

    private function sendWelcomeEmail(User $user): void
    {
        // Your welcome email logic...
        // $this->mailer->send(...);
    }
}
```

The `Autowire` attribute is used to inject the built-in processor services registered by API Platform.

If you're using Doctrine MongoDB ODM instead of Doctrine ORM, replace `orm` by `odm` in the name of the injected services.

Finally, configure that you want to use this processor on the User resource:

```php
<?php
// api/src/Entity/User.php

namespace App\Entity;

use ApiPlatform\Metadata\ApiResource;
use App\State\UserProcessor;

#[ApiResource(processor: UserProcessor::class)]
class User {}
```

### Laravel State Provider mechanism
If you want to execute custom business logic before or after persistence, this can be achieved by using [composition](https://en.wikipedia.org/wiki/Object_composition).

Here is an implementation example which uses [Laravel Mail](https://laravel.com/docs/mail) to send new users a welcome email after a REST `POST` or GraphQL `create` operation, in a project using the native Eloquent ORM state processor:

```php
<?php
// api/app/State/UserProcessor.php

namespace App\State;

use ApiPlatform\Metadata\DeleteOperationInterface;
use ApiPlatform\Metadata\Operation;
use ApiPlatform\State\ProcessorInterface;
use App\Models\User;
use Symfony\Component\DependencyInjection\Attribute\Autowire;
use Symfony\Component\Mailer\MailerInterface;

/**
 * @implements ProcessorInterface<User, User|void>
 */
final class UserProcessor implements ProcessorInterface
{
    public function __construct(
        private ProcessorInterface $persistProcessor,
        private ProcessorInterface $removeProcessor,
    )
    {
    }

    /**
     * @return User|void
     */
    public function process(mixed $data, Operation $operation, array $uriVariables = [], array $context = []): mixed
    {
        if ($operation instanceof DeleteOperationInterface) {
            return $this->removeProcessor->process($data, $operation, $uriVariables, $context);
        }
    
        $result = $this->persistProcessor->process($data, $operation, $uriVariables, $context);
        $this->sendWelcomeEmail($data);

        return $result;
    }

    private function sendWelcomeEmail(User $user): void
    {
        // Your welcome email logic...
        // Mail::to($user->getEmail())->send(new WelcomeMail($user));
    }
}
```

Don't forget to tag the service with the `PersistProcessor` and the `RemoveProcessor` state classes.

```php
<?php

namespace App\Providers;

use ApiPlatform\Laravel\Eloquent\State\PersistProcessor;
use ApiPlatform\Laravel\Eloquent\State\RemoveProcessor;
use App\State\UserProcessor;
use Illuminate\Support\ServiceProvider;

class AppServiceProvider extends ServiceProvider
{
    public function register(): void
    {
        $this->app->tag([UserProcessor::class], [PersistProcessor::class, RemoveProcessor::class,]);
    }

    public function boot(): void
    {
    }
}
```
If you're using Laravel MongoDB ODM instead of Eloquent ORM, make sure you're using the right services.

Finally, configure that you want to use this processor on the User resource:

```php
<?php
// api/app/Models/User.php

namespace App\Models;

use ApiPlatform\Metadata\ApiResource;
use App\State\UserProcessor;

#[ApiResource(processor: UserProcessor::class)]
class User {}
```

## Registering Services Without Autowiring (only for the Symfony variant)

The previous examples work because service autowiring and autoconfiguration are enabled by default in Symfony and API Platform.
If you disabled this feature, you need to register the services by yourself and add the `api_platform.state_processor` tag.

```yaml
# api/config/services.yaml

services:
    # ...
    App\State\BlogPostProcessor: ~
        tags: [ 'api_platform.state_processor' ]

    App\State\UserProcessor:
        arguments:
            $persistProcessor: '@api_platform.doctrine.orm.state.persist_processor'
            $removeProcessor: '@api_platform.doctrine.orm.state.remove_processor'
            # If you're using Doctrine MongoDB ODM, you can use the following code:
            # $persistProcessor: '@api_platform.doctrine_mongodb.odm.state.persist_processor'
            # $removeProcessor: '@api_platform.doctrine_mongodb.odm.state.remove_processor'
            $mailer: '@mailer'
        tags: [ 'api_platform.state_processor' ]
```

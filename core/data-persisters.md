# Data Persisters

To mutate the application states during `POST`, `PUT`, `PATCH` or `DELETE` [operations](operations.md), API Platform uses
classes called **data persisters**. Data persisters receive an instance of the class marked as an API resource (usually using
the `#[ApiResource]` attribute). This instance contains data submitted by the client during [the deserialization
process](serialization.md).

<p align="center" class="symfonycasts"><a href="https://symfonycasts.com/screencast/api-platform-security/encode-user-password?cid=apip"><img src="../distribution/images/symfonycasts-player.png" alt="Data Persister screencast"><br>Watch the Data Persister screencast</a></p>

A data persister using [Doctrine ORM](https://www.doctrine-project.org/projects/orm.html) is included with the library and
is enabled by default. It is able to persist and delete objects that are also mapped as [Doctrine entities](https://www.doctrine-project.org/projects/doctrine-orm/en/current/reference/basic-mapping.html).
A [Doctrine MongoDB ODM](https://www.doctrine-project.org/projects/mongodb-odm.html) data persister is also included and can be enabled by following the [MongoDB documentation](mongodb.md).

However, you may want to:

* store data to other persistence layers (Elasticsearch, external web services...)
* not publicly expose the internal model mapped with the database through the API
* use a separate model for [read operations](data-providers.md) and for updates by implementing patterns such as [CQRS](https://martinfowler.com/bliki/CQRS.html)

Custom data persisters can be used to do so. A project can include as many data persisters as needed. The first able to
persist data for a given resource will be used.

## Creating a Custom Data Persister

To create a data persister, you have to implement the [`ContextAwareDataPersisterInterface`](https://github.com/api-platform/core/blob/2.6/src/DataPersister/ContextAwareDataPersisterInterface.php).
This interface defines only 3 methods:

* `persist`: to create or update the given data
* `remove`: to delete the given data
* `supports`: to check whether the given data is supported by this data persister

Here is an implementation example:

```php
namespace App\DataPersister;

use ApiPlatform\Core\DataPersister\ContextAwareDataPersisterInterface;
use App\Entity\BlogPost;

final class BlogPostDataPersister implements ContextAwareDataPersisterInterface
{
    public function supports($data, array $context = []): bool
    {
        return $data instanceof BlogPost;
    }

    public function persist($data, array $context = [])
    {
        // call your persistence layer to save $data
        return $data;
    }

    public function remove($data, array $context = [])
    {
        // call your persistence layer to delete $data
    }
}
```

If service autowiring and autoconfiguration are enabled (they are by default), you are done!

Otherwise, if you use a custom dependency injection configuration, you need to register the corresponding service and add the
`api_platform.data_persister` tag. The `priority` attribute can be used to order persisters.

```yaml
# config/services.yaml
services:
    # ...
    App\DataPersister\BlogPostDataPersister: ~
        # Uncomment only if autoconfiguration is disabled
        #tags: [ 'api_platform.data_persister' ]
```

Note that if you don't need any `$context` in your data persister's methods, you can implement the [`DataPersisterInterface`](https://github.com/api-platform/core/blob/2.6/src/DataPersister/DataPersisterInterface.php) instead.

## Decorating the Built-In Data Persisters

<p align="center" class="symfonycasts"><a href="https://symfonycasts.com/screencast/api-platform-extending/persister-decoration?cid=apip"><img src="../distribution/images/symfonycasts-player.png" alt="Data Persister Decoration screencast"><br>Watch the Data Persister Decoration screencast</a></p>

If you want to execute custom business logic before or after persistence, this can be achieved by [decorating](https://symfony.com/doc/current/service_container/service_decoration.html) the built-in data persisters.

The next example uses [Symfony Mailer](https://symfony.com/doc/current/mailer.html). Read its documentation if you want to use it.

Here is an implementation example which sends new users a welcome email after a REST `POST` or GraphQL `create` operation, in a project using the native Doctrine ORM data persister:

```php
namespace App\DataPersister;

use ApiPlatform\Core\DataPersister\ContextAwareDataPersisterInterface;
use App\Entity\User;
use Symfony\Component\Mailer\MailerInterface;

final class UserDataPersister implements ContextAwareDataPersisterInterface
{
    private $decorated;
    private $mailer;

    public function __construct(ContextAwareDataPersisterInterface $decorated, MailerInterface $mailer)
    {
        $this->decorated = $decorated;
        $this->mailer = $mailer;
    }

    public function supports($data, array $context = []): bool
    {
        return $this->decorated->supports($data, $context);
    }

    public function persist($data, array $context = [])
    {
        $result = $this->decorated->persist($data, $context);

        if (
            $data instanceof User && (
                ($context['collection_operation_name'] ?? null) === 'post' ||
                ($context['graphql_operation_name'] ?? null) === 'create'
            )
        ) {
            $this->sendWelcomeEmail($data);
        }

        return $result;
    }

    public function remove($data, array $context = [])
    {
        return $this->decorated->remove($data, $context);
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
# config/services.yaml
services:
    # ...
    App\DataPersister\UserDataPersister:
        bind:
            $decorated: '@api_platform.doctrine.orm.data_persister'
        # Uncomment only if autoconfiguration is disabled
        #arguments: ['@App\DataPersister\UserDataPersister.inner']
        #tags: [ 'api_platform.data_persister' ]
```

## Calling multiple DataPersisters

Our DataPersisters are called in chain, once a data persister is supported the chain breaks and API Platform assumes your data is persisted. You can call multiple data persisters by implementing the `ResumableDataPersisterInterface`:

```php
namespace App\DataPersister;

use ApiPlatform\Core\DataPersister\ContextAwareDataPersisterInterface;
use App\Entity\BlogPost;

final class BlogPostDataPersister implements ContextAwareDataPersisterInterface, ResumableDataPersisterInterface 
{
    public function supports($data, array $context = []): bool
    {
        return $data instanceof BlogPost;
    }

    public function persist($data, array $context = [])
    {
        // call your persistence layer to save $data
        return $data;
    }

    public function remove($data, array $context = [])
    {
        // call your persistence layer to delete $data
    }

    // Once called this data persister will resume to the next one
    public function resumable(array $context = []): bool 
    {
        return true;
    }
}
```

This is useful when using [`Messenger` with API Platform](messenger.md) as you may want to do something asynchronously with the data but still call the default Doctrine data persister, for example:

```php
namespace App\DataPersister;

use ApiPlatform\Core\DataPersister\ContextAwareDataPersisterInterface;
use Doctrine\ORM\EntityManagerInterface;
use App\Entity\BlogPost;

final class BlogPostDataPersister implements ContextAwareDataPersisterInterface, ResumableDataPersisterInterface 
{
    private $entityManager;
    
    public function __construct(EntityManagerInterface $entityManager)
    {
        $this->entityManager = $entityManager;
    }
    
    public function supports($data, array $context = []): bool
    {
        return $data instanceof BlogPost;
    }

    public function persist($data, array $context = [])
    {
        $this->entityManager->persist($data);
        $this->entityManager->flush();
    }

    public function remove($data, array $context = [])
    {
        $this->entityManager->remove($data);
        $this->entityManager->flush();
    }

    // Once called this data persister will resume to the next one
    public function resumable(array $context = []): bool 
    {
        return true;
    }
}
```

```yaml
# config/services.yaml
services:
    # ...
    App\DataPersister\BlogPostDataPersister: ~
        # Uncomment only if autoconfiguration is disabled
        #arguments: ['@App\DataPersister\BlogPostDataPersister.inner']
        #tags: [ 'api_platform.data_persister' ]
```

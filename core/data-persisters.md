# Data Persisters

To mutate the application states during `POST`, `PUT`, `PATCH` or `DELETE` [operations](operations.md), API Platform uses
classes called **data persisters**. Data persisters receive an instance of the class marked as an API resource (usually using
the `@ApiResource` annotation). This instance contains data submitted by the client during [the deserialization
process](serialization.md).

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

To create a data persister, you have to implement the [`ContextAwareDataPersisterInterface`](https://github.com/api-platform/core/blob/master/src/DataPersister/ContextAwareDataPersisterInterface.php).
You will need to avoid recursion by using the context to let the persister know it was already called when going through the chain of persisters. Using $context[self::ALREADY_CALLED] = true in persist and remove. And adding a check in the support method.
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
    private const ALREADY_CALLED = 'BLOG_POST_PERSISTER_ALREADY_CALLED';
    
    public function supports($data, array $context = []): bool
    {
        if (isset($context[self::ALREADY_CALLED])) {
            return false;
        }    
        return $data instanceof BlogPost;
    }

    public function persist($data, array $context = [])
    {
      $context[self::ALREADY_CALLED] = true;    
      // call your persistence layer to save $data
      return $data;
    }

    public function remove($data, array $context = [])
    {
      $context[self::ALREADY_CALLED] = true;    
      // call your persistence layer to delete $data
    }
}
```

If service autowiring and autoconfiguration are enabled (they are by default), you are done!

Otherwise, if you use a custom dependency injection configuration, you need to register the corresponding service and add the
`api_platform.data_persister` tag. The `priority` attribute can be used to order persisters.

```yaml
# api/config/services.yaml
services:
    # ...
    App\DataPersister\BlogPostDataPersister: ~
        # Uncomment only if autoconfiguration is disabled
        #tags: [ 'api_platform.data_persister' ]
```

Note that if you don't need any `$context` in your data persister's methods, you can implement the [`DataPersisterInterface`](https://github.com/api-platform/core/blob/master/src/DataPersister/DataPersisterInterface.php) instead.

## Decorating the Built-In Data Persisters

If you want to execute custom business logic before or after peristence, this can be achieved by [decorating](https://symfony.com/doc/current/service_container/service_decoration.html) the built-in data persisters.

Here is an implementation example which sends new users a welcome email after a REST `POST` or GraphQL `create` operation, in a project using the native Doctrine ORM data persister: Again here don't forget to use the context to avoid recursion.

```php
namespace App\DataPersister;

use ApiPlatform\Core\DataPersister\DataPersisterInterface;
use ApiPlatform\Core\DataPersister\ContextAwareDataPersisterInterface;
use App\Entity\User;
use Symfony\Component\Mailer\MailerInterface;

final class UserDataPersister implements ContextAwareDataPersisterInterface
{
    private const ALREADY_CALLED = 'BLOG_POST_PERSISTER_ALREADY_CALLED';
    private $decorated;
    private $mailer;

    public function __construct(DataPersisterInterface $decorated, MailerInterface $mailer)
    {
        $this->decorated = $decorated;
        $this->mailer = $mailer;
    }

    public function supports($data, array $context = []): bool
    {
        if (isset($context[self::ALREADY_CALLED])) {
            return false;
        } 
        return $data instanceof User;
    }

    public function persist($data, array $context = [])
    {
        $context[self::ALREADY_CALLED] = true;    
        $result = $this->decorated->persist($data, $context);

        if (
            ($context['collection_operation_name'] ?? null) === 'post' ||
            ($context['graphql_operation_name'] ?? null) === 'create'
        ) {
            $this->sendWelcomeMail($data);
        }

        return $result;
    }

    public function remove($data, array $context = [])
    {
        $context[self::ALREADY_CALLED] = true;    
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
# api/config/services.yaml
services:
    # ...
    App\DataPersister\UserDataPersister:
        decorates: 'api_platform.doctrine.orm.data_persister'
        # Uncomment only if autoconfiguration is disabled
        #arguments: ['@App\DataPersister\UserDataPersister.inner']
        #tags: [ 'api_platform.data_persister' ]
```

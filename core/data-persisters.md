# Data Persisters

To mutate the application states during `POST`, `PUT`, `PATCH` or `DELETE` [operations](operations.md), API Platform uses
classes called **data persisters**. Data persisters receive an instance of the class marked as an API resource (usually using
the `@ApiResource` annotation). This instance contains data submitted by the client during [the deserialization
process](serialization.md).

A data persister using [Doctrine ORM](http://www.doctrine-project.org/projects/orm.html) is included with the library and
is enabled by default. It is able to persist and delete objects that are also mapped as [Doctrine entities](https://www.doctrine-project.org/projects/doctrine-orm/en/2.6/reference/basic-mapping.html).

However, you may want to:

* store data to other persistence layers (ElasticSearch, MongoDB, external web services...)
* not publicly expose the internal model mapped with the database through the API
* use a separate model for [read operations](data-providers.md) and for updates by implementing patterns such as [CQRS](https://martinfowler.com/bliki/CQRS.html)

Custom data persisters can be used to do so. A project can include as many data persisters as it needs. The first able to
persist data for a given resource will be used.

## Creating a Custom Data Persister

To create a data persister, you have to implement the [`DataPersisterInterface`](https://github.com/api-platform/core/blob/master/src/DataPersister/DataPersisterInterface.php).
This interface defines only 3 methods:

* `persist`: to create or update the given data
* `delete`: to delete the given data
* `support`: to tell if the given data can be handled by this specific data persister

Here is an implementation example:

```php
namespace App\DataPersister;

use ApiPlatform\Core\DataPersister\DataPersisterInterface;
use App\Entity\BlogPost;

final class BlogPostDataPersister implements DataPersisterInterface
{
    public function supports($data): bool
    {
        return $data instanceof BlogPost;
    }
    
    public function persist($data)
    {
      // call your persistence layer to save $data
      return $data;
    }
    
    public function delete($data): void
    {
      // call your persistence layer to delete $data
    }
}
```

If service autowiring and autoconfiguration are enabled (it's the case by default), you are done!

Otherwise, if you use a custom dependency injection configuration, you need to register the corresponding service and add the
`api_platform.data_persister` tag to it.  The `priority` attribute can be used to order persisters.

```yaml
# api/config/services.yaml
services:
    # ...
    'App\DataPersister\BlogPostDataPersister': ~
        # Uncomment only if autoconfiguration is disabled
        #tags: [ 'api_platform.data_persister' ]
```

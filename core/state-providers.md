# State Providers

To retrieve data exposed by the API, API Platform uses classes called **state providers**. A state provider using [Doctrine
ORM](https://www.doctrine-project.org/projects/orm.html) to retrieve data from a database, a state provider using
[Doctrine MongoDB ODM](https://www.doctrine-project.org/projects/mongodb-odm.html) to retrieve data from a document
database, and a state provider using [Elasticsearch-PHP](https://www.elastic.co/guide/en/elasticsearch/client/php-api/current/index.html)
to retrieve data from an Elasticsearch cluster are included with the library. The first one is enabled by default. These
state providers natively support paged collections and filters. They can be used as-is and are perfectly suited to common uses.

However, you sometimes want to retrieve data from other sources such as another persistence layer or a webservice.
Custom state providers can be used to do so. A project can include as many state providers as needed. The first able to
retrieve data for a given resource will be used.

To do so you need to implement the `ApiPlatform\State\ProviderInterface`.

In the following examples we will create custom data providers for an entity class called `App\Entity\BlogPost`.
Note, that if your entity is not Doctrine-related, you need to flag the identifier property by using
`#[ApiProperty(identifier: true)` for things to work properly (see also [Entity Identifier Case](serialization.md#entity-identifier-case)).

## State Provider

If the [Symfony MakerBundle](https://symfony.com/doc/current/bundles/SymfonyMakerBundle) is installed in your project,
you can use the following command to generate a custom state provider easily:

```console
bin/console make:state-provider
```

Let's start with a State Provider for the URI: `/blog_posts/{id}`, which operation name is `_api_/blog_posts/{id}_get`.
You can find this information either with the `debug:router` command (the route name and the operation name are the same),
or by using the `debug:api` command.

First, your `BlogPostStateProvider` has to implement the
[`StateProviderInterface`](https://github.com/api-platform/core/blob/main/src/State/StateProviderInterface.php):

```php
<?php

namespace App\State;

use App\Entity\BlogPost;
use ApiPlatform\State\ProviderInterface;

final class BlogPostStateProvider implements ProviderInterface
{
    /**
     * Provides data.
     *
     * @return object|array|null
     */
    public function provide(string $resourceClass, array $uriVariables = [], ?string $operationName = null, array $context = [])
    {
        return new BlogPost($uriVariables['id']);
    }

    /**
     * Whether this state provider supports the class/identifier tuple.
     */
    public function supports(string $resourceClass, array $uriVariables = [], ?string $operationName = null, array $context = []): bool
    {
        return BlogPost::class === $resourceClass && '_api_/blog_posts/{id}_get' === $operationName;
    }
}
```

In the `supports` method, we declare that this State Provider only works for the given operation. As this operation expects a
BlogPost we return an instance of the BlogPost in the `provide` method.
The `uriVariables` parameter is an array with the values of the URI variables.

Now let's say that we also want to handle the `/blog_posts` URI which returns a collection. We can change the Provider into
supporting a wider range of operations. Then we can provide a collection of blog posts when the operation is a `GetCollection`:

```php
<?php

namespace App\State;

use App\Entity\BlogPost;
use ApiPlatform\State\ProviderInterface;
use ApiPlatform\Metadata\GetCollection;

final class BlogPostStateProvider implements ProviderInterface
{
    /**
     * Provides data.
     *
     * @return object|array|null
     */
    public function provide(string $resourceClass, array $uriVariables = [], ?string $operationName = null, array $context = [])
    {
        if ($context['operation'] instanceof GetCollection) {
            return [new BlogPost(), new BlogPost()];
        }

        return new BlogPost($uriVariables['id']);
    }

    /**
     * Whether this state provider supports the class/identifier tuple.
     */
    public function supports(string $resourceClass, array $uriVariables = [], ?string $operationName = null, array $context = []): bool
    {
        return $data instanceof BlogPost;
    }
}
```

If you use the default configuration, the corresponding service will be automatically registered thanks to
[autowiring](https://symfony.com/doc/current/service_container/autowiring.html).
To declare the service explicitly, or to set a custom priority, you can use the following snippet:

```yaml
# api/config/services.yaml
services:
    # ...
    'App\DataProvider\BlogPostStateProvider':
        tags: [ { name: 'api_platform.state_provider', priority: 2 } ]
        # Autoconfiguration must be disabled to set a custom priority
        autoconfigure: false
```

Tagging the service with the tag `api_platform.state_provider` will enable API Platform Core to automatically
register and use this state provider. The optional attribute `priority` allows you to define the order in which the
data providers are called.

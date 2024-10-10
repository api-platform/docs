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

In the following examples we will create custom state providers for an entity class called `App\Entity\BlogPost`.
Note, that if your entity is not Doctrine-related, you need to flag the identifier property by using
`#[ApiProperty(identifier: true)` for things to work properly (see also [Entity Identifier Case](serialization.md#entity-identifier-case)).

## Creating a Custom State Provider

If the [Symfony MakerBundle](https://symfony.com/doc/current/bundles/SymfonyMakerBundle) is installed in your project,
you can use the following command to generate a custom state provider easily:

```console
bin/console make:state-provider
```

Let's start with a State Provider for the URI: `/blog_posts/{id}`.

First, your `BlogPostProvider` has to implement the
[`ProviderInterface`](https://github.com/api-platform/core/blob/main/src/State/ProviderInterface.php):

```php
<?php
// api/src/State/BlogPostProvider.php

namespace App\State;

use App\Entity\BlogPost;
use ApiPlatform\Metadata\Operation;
use ApiPlatform\State\ProviderInterface;

/**
 * @implements ProviderInterface<BlogPost|null>
 */
final class BlogPostProvider implements ProviderInterface
{
    private const DATA = [
        'ab' => new BlogPost('ab'),
        'cd' => new BlogPost('cd'),
    ];

    public function provide(Operation $operation, array $uriVariables = [], array $context = []): BlogPost|null
    {
        return self::DATA[$uriVariables['id']] ?? null;
    }
}
```

For the example, we store the list of our blog posts in an associative array (the `BlogPostProvider::DATA` constant).

As this operation expects a `BlogPost`, the `provide` methods return the instance of the `BlogPost` corresponding to the ID passed in the URL. If the ID doesn't exist in the associative array, `provide()` returns `null`. API Platform will automatically generate a 404 response if the provider returns `null`.

The `$uriVariables` parameter contains an array with the values of the URI variables.

To use this provider we need to configure the provider on the operation:

```php
<?php
// api/src/Entity/BlogPost.php

namespace App\Entity;

use ApiPlatform\Metadata\Get;
use App\State\BlogPostProvider;

#[Get(provider: BlogPostProvider::class)]
class BlogPost {}
```

Now let's say that we also want to handle the `/blog_posts` URI which returns a collection. We can change the Provider into
supporting a wider range of operations. Then we can provide a collection of blog posts when the operation is a `CollectionOperationInterface`:

```php
<?php
// api/src/State/BlogPostProvider.php

namespace App\State;

use App\Entity\BlogPost;
use ApiPlatform\Metadata\Operation;
use ApiPlatform\State\ProviderInterface;
use ApiPlatform\Metadata\CollectionOperationInterface;

/**
 * @implements ProviderInterface<BlogPost[]|BlogPost|null>
 */
final class BlogPostProvider implements ProviderInterface
{
    private const DATA = [
        'ab' => new BlogPost('ab'),
        'cd' => new BlogPost('cd'),
    ];

    public function provide(Operation $operation, array $uriVariables = [], array $context = []): iterable|BlogPost|null
    {
        if ($operation instanceof CollectionOperationInterface) {
            return self::DATA;
        }

        return self::DATA[$uriVariables['id']] ?? null;
    }
}
```

We then need to configure this same provider on the BlogPost `GetCollection` operation, or for every operations via the `ApiResource` attribute:

```php
<?php
// api/src/Entity/BlogPost.php

namespace App\Entity;

use ApiPlatform\Metadata\ApiResource;
use App\State\BlogPostProvider;

#[ApiResource(provider: BlogPostProvider::class)]
class BlogPost {}
```

## Hooking into the Built-In State Provider

If you want to execute custom business logic before or after retrieving data, this can be achieved by [decorating](https://symfony.com/doc/current/service_container/service_decoration.html) the built-in state providers or using [composition](https://en.wikipedia.org/wiki/Object_composition).

The next example uses a [DTO](https://api-platform.com/docs/core/dto/#using-data-transfer-objects-dtos) to change the presentation for data originally retrieved by the default state provider.

```php
<?php
// api/src/State/BlogPostProvider.php

namespace App\State;

use App\Dto\AnotherRepresentation;
use App\Model\Book;
use ApiPlatform\Metadata\Operation;
use ApiPlatform\State\ProviderInterface;
use Symfony\Component\DependencyInjection\Attribute\Autowire;

/**
 * @implements ProviderInterface<AnotherRepresentation>
 */
final class BookRepresentationProvider implements ProviderInterface
{
    public function __construct(
        #[Autowire(service: 'api_platform.doctrine.orm.state.item_provider')]
        private ProviderInterface $itemProvider,
    )
    {
    }

    public function provide(Operation $operation, array $uriVariables = [], array $context = []): AnotherRepresentation
    {
        $book = $this->itemProvider->provide($operation, $uriVariables, $context);

        return new AnotherRepresentation(
            // Add DTO constructor params here.
            // $book->getTitle(),
        );
    }
}
```

And configure that you want to use this provider on the Book resource:

```php
<?php
// api/src/Entity/Book.php

namespace App\Entity;

use ApiPlatform\Metadata\Get;
use App\Dto\AnotherRepresentation;
use App\State\BookRepresentationProvider;

#[Get(output: AnotherRepresentation::class, provider: BookRepresentationProvider::class)]
class Book {}
```

## Registering Services Without Autowiring

The services in the previous examples are automatically registered because
[autowiring](https://symfony.com/doc/current/service_container/autowiring.html)
and autoconfiguration are enabled by default in API Platform.
To declare the service explicitly, you can use the following snippet:

```yaml
# api/config/services.yaml

services:
    # ...
    App\State\BlogPostProvider: ~
        tags: [ 'api_platform.state_provider' ]

# api/config/services.yaml
services:
    # ...
    App\State\BookRepresentationProvider:
        arguments:
            $itemProvider: '@api_platform.doctrine.orm.state.item_provider'
        tags: [ 'api_platform.state_provider' ]
```

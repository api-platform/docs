# GraphQL Support

[GraphQL](https://graphql.org/) is a query language made to communicate with an API and therefore is an alternative to REST.

It has some advantages compared to REST: it solves the over-fetching or under-fetching of data, is strongly typed, and is capable of retrieving multiple and nested data in one go, but it also comes with drawbacks. For example it creates overhead depending on the request.

API Platform creates a REST API by default. But you can choose to enable GraphQL as well.

Once enabled, you have nothing to do: your schema describing your API is automatically built and your GraphQL endpoint is ready to go!

## Enabling GraphQL

To enable GraphQL and its IDE (GraphiQL and GraphQL Playground) in your API, simply require the `api-platform/graphql` package using Composer:

```console
composer require api-platform/graphql
```

You can now use GraphQL at the endpoint: `https://localhost:8443/graphql`.

> [!NOTE]
> If you used [the Symfony Variant thanks to Symfony Flex](../symfony/index.md#installing-the-framework) or the Laravel
> variant, URLs will be prefixed with `/api` by default. For example, the GraphQL endpoint will be: `https://localhost:8443/api/graphql`.

## Changing Location of the GraphQL Endpoint

Sometimes you may want to have the GraphQL endpoint at a different location. This can be done by manually configuring the GraphQL controller.

### Symfony Routes

Using the Symfony variant we can do this modification by adding the following code:

```yaml
# api/config/routes.yaml
api_graphql_entrypoint:
  path: /api/graphql
  controller: api_platform.graphql.action.entrypoint
# ...
```

Change `/api/graphql` to the URI you wish the GraphQL endpoint to be accessible on.

### Laravel Routes

Using the Laravel variant we can do this modification by adding the following code:

```php
// routes/web.php
use Illuminate\Support\Facades\Route;
use ApiPlatform\GraphQL\Action\EntrypointAction;

Route::post('/api/graphql', EntrypointAction::class)
    ->name('api_graphql_entrypoint');
```

Change `/api/graphql` to the URI you wish the GraphQL endpoint to be accessible on.

## GraphiQL

Go to the GraphQL endpoint with your browser, you will see a nice interface provided by GraphiQL to interact with your API.

The GraphiQL IDE can also be found at `/graphql/graphiql`.

If you need to disable it, it can be done in the configuration:

### Disabling GraphiQL with Symfony

```yaml
# api/config/packages/api_platform.yaml
api_platform:
  graphql:
    graphiql:
      enabled: false
# ...
```

### Disabling GraphiQL with Laravel

```php
<?php
// config/api-platform.php
return [
    // ....
    'graphql' => [
        'graphiql' => [
            'enabled' => false,
        ]
    ],
];
```

### Add another Location for GraphiQL

Sometimes you may want to have the GraphiQL at a different location. This can be done by manually configuring the GraphiQL controller.

### Symfony config routes for GraphiQL

If you want to add a different location besides `/graphql/graphiql`, you can do it like this if you are using the Symfony variant:

```yaml
# app/config/routes.yaml
graphiql:
  path: /docs/graphiql
  controller: api_platform.graphql.action.graphiql
```

### Laravel config routes for GraphiQL

If you want to add a different location besides `/graphql/graphiql`, you can do it like this if you are using the Laravel variant:

```php
// routes/web.php
use Illuminate\Support\Facades\Route;
use ApiPlatform\GraphQL\Action\GraphiQlAction;

Route::post('/docs/graphiql', GraphiQlAction::class)
    ->name('graphiql');
```

## GraphQL Playground

Another IDE is by default included in API Platform: GraphQL Playground.

It can be found at `/graphql/graphql_playground`.

You can disable it if you want in the configuration.

### Disable GraphQL Playground with Symfony

```yaml
# api/config/packages/api_platform.yaml
api_platform:
  graphql:
    graphql_playground:
      enabled: false
# ...
```

### Disable GraphQL Playground with Laravel

> [!WARNING]
> This is not yet available with Laravel, you're welcome to contribute [on GitHub](https://github.com/api-platform/core)

### Add another Location for GraphQL Playground

You can add a different location besides `/graphql/graphql_playground`.

### Symfony config routes for GraphQL Playground

Using the Symfony variant we can do this modification by adding the following code:

```yaml
# app/config/routes.yaml
graphql_playground:
  path: /docs/graphql_playground
  controller: api_platform.graphql.action.graphql_playground
```

### Laravel config routes for GraphQL Playground

Using the Laravel variant we can do this modification by adding the following code:

```php
// routes/web.php
use Illuminate\Support\Facades\Route;
use ApiPlatform\GraphQL\Action\GraphQlPlaygroundAction;

Route::post('/docs/graphql_playground', GraphQlPlaygroundAction::class)
    ->name('graphql_playground');
```

## Modifying or Disabling the Default IDE

When going to the GraphQL endpoint, you can choose to launch the IDE you want.

### Symfony config to modifying the default IDE

```yaml
# api/config/packages/api_platform.yaml
api_platform:
  graphql:
    # Choose between graphiql or graphql-playground
    default_ide: graphql-playground
# ...
```

### Laravel config to modifying the default IDE

```php
<?php
// config/api-platform.php
return [
    // ....
    'graphql' => [
        // Choose between graphiql or graphql-playground
        'default_ide' => 'graphql-playground',
    ],
];
```

You can also disable this feature by setting the configuration value to `false`.

### Symfony config to disable default IDE

```yaml
# api/config/packages/api_platform.yaml
api_platform:
  graphql:
    default_ide: false
# ...
```

### Laravel config to disable default IDE

```php
<?php
// config/api-platform.php
return [
    // ....
    'graphql' => [
        'default_ide' => false,
    ],
];
```

## Disabling the Introspection Query

For security reason, the introspection query should be disabled to not expose the GraphQL schema.

### Symfony config to disable the Introspection Query

If you need to disable it, it can be done in the configuration:

```yaml
# api/config/packages/api_platform.yaml
api_platform:
  graphql:
    introspection: false
# ...
```

### Laravel config to disable the Introspection Query

If you need to disable it, it can be done in the configuration:

```php
<?php
// config/api-platform.php
return [
    // ....
    'graphql' => [
        'introspection' => false,
    ],
];
```

## Request with `application/graphql` Content-Type

If you wish to send a [POST request using the `application/graphql` Content-Type](https://graphql.org/learn/serving-over-http/#post-request),
you need to enable it in the [allowed formats of API Platform](content-negotiation.md#configuring-formats-globally):

### Symfony config for GraphQL Content-Type

```yaml
# api/config/packages/api_platform.yaml
api_platform:
  formats:
    # ...
    graphql: ['application/graphql']
```

### Laravel config for GraphQL Content-Type

```php
<?php
// config/api-platform.php
return [
    // ....
    'formats' => [
        'graphql' => [
            'application/graphql',
        ],
    ],
];
```

## Operations

> [!NOTE]
> In Symfony we use the term “entities”, while the following documentation is mostly for Laravel “models”.

To understand what an operation is, please refer to the [operations documentation](operations.md).

For GraphQL, the operations are defined by using the `Query`, `QueryCollection`, `Mutation`, `DeleteMutation` and `Subscription` attributes.

By default, the following operations are enabled:

- `Query`
- `QueryCollection`
- `Mutation(name: 'create')`
- `Mutation(name: 'update')`
- `DeleteMutation(name: 'delete')`

You can of course disable or configure these operations.

For instance, in the following example, only the query of an item and the create mutation are enabled:

<code-selector>

```php
<?php
// api/src/Entity/Book.php
namespace App\Entity;

use ApiPlatform\Metadata\GraphQl\Mutation;
use ApiPlatform\Metadata\GraphQl\Query;
use ApiPlatform\Metadata\ApiResource;

#[ApiResource(graphQlOperations: [
    new Query(),
    new Mutation(name: 'create')
])]
class Book
{
    // ...
}
```

```yaml
#The YAML syntax is only supported for Symfony
resources:
  App\Entity\Book:
    graphQlOperations:
      ApiPlatform\Metadata\GraphQl\Query: ~
      ApiPlatform\Metadata\GraphQl\Mutation:
        name: create
```

```xml
<!--The XML syntax is only supported for Symfony -->
<resources xmlns="https://api-platform.com/schema/metadata/resources-3.0"
           xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
           xsi:schemaLocation="https://api-platform.com/schema/metadata/resources-3.0
           https://api-platform.com/schema/metadata/resources-3.0.xsd">
    <resource class="App\Entity\Book">
        <graphQlOperations>
            <graphQlOperation class="ApiPlatform\Metadata\GraphQl\Query" />
            <graphQlOperation class="ApiPlatform\Metadata\GraphQl\Mutation" name="create" />
        </graphQlOperations>
    </resource>
</resources>
```

</code-selector>

## Queries

If you don't know what queries are yet, please [read the documentation about them](https://graphql.org/learn/queries/).

For each resource, two queries are available: one for retrieving an item and the other one for the collection.
For example, if you have a `Book` resource, the queries `book` and `books` can be used.

### Global Object Identifier

When querying an item, you need to pass an identifier as argument. Following the [GraphQL Global Object Identification Specification](https://relay.dev/graphql/objectidentification.htm),
the identifier needs to be globally unique. In API Platform, this argument is represented as an [IRI (Internationalized Resource Identifier)](https://www.w3.org/TR/ld-glossary/#internationalized-resource-identifier).

For example, to query a book having as identifier `89`, you have to run the following:

```graphql
{
  book(id: "/books/89") {
    title
    isbn
  }
}
```

Note that in this example, we're retrieving two fields: `title` and `isbn`.

### Custom Queries

To create a custom query, first of all you need to create its resolver.

If you want a custom query for a collection, create a class like this:

```php
<?php
// api/src/Resolver/BookCollectionResolver.php
namespace App\Resolver;

use ApiPlatform\GraphQl\Resolver\QueryCollectionResolverInterface;
use App\Entity\Book;

final class BookCollectionResolver implements QueryCollectionResolverInterface
{
    /**
     * @param iterable<Book> $collection
     *
     * @return iterable<Book>
     */
    public function __invoke(iterable $collection, array $context): iterable
    {
        // Query arguments are in $context['args'].

        foreach ($collection as $book) {
            // Do something with the book.
        }

        return $collection;
    }
}
```

### Custom Queries config for Symfony

If you use autoconfiguration (the default Symfony configuration) in your application, then you are done!

Else, you need to tag your resolver like this if you are using Symfony without autoconfiguration :

```yaml
# api/config/services.yaml
services:
  # ...
  App\Resolver\BookCollectionResolver:
    tags:
      - { name: api_platform.graphql.query_resolver }
```

### Custom Queries config for Laravel

If you are using Laravel tag your resolver with:

```php
<?php

namespace App\Providers;

use ApiPlatform\GraphQl\Resolver\QueryItemResolverInterface;
use App\Resolver\BookCollectionResolver;
use Illuminate\Support\ServiceProvider;

class AppServiceProvider extends ServiceProvider
{
    public function register(): void
    {
        $this->app->tag([BookCollectionResolver::class], QueryItemResolverInterface::class);
    }

    public function boot(): void
    {
    }
}
```

The resolver for an item is very similar:

```php
<?php
// api/src/Resolver/BookResolver.php
namespace App\Resolver;

use ApiPlatform\GraphQl\Resolver\QueryItemResolverInterface;
use App\Entity\Book;

final class BookResolver implements QueryItemResolverInterface
{
    /**
     * @param Book|null $item
     *
     * @return Book
     */
    public function __invoke($item, array $context)
    {
        // Query arguments are in $context['args'].

        // Do something with the book.
        // Or fetch the book if it has not been retrieved.

        return $item;
    }
}
```

Note that you will receive the retrieved item or not in this resolver depending on how you configure your query in your resource.

Since the resolver is a service, you can inject some dependencies and fetch your item in the resolver if you want.

Now that your resolver is created and registered, you can configure your custom query and link its resolver.

In your resource, add the following:

<code-selector>

```php
<?php
// api/src/Entity/Book.php
namespace App\Entity;

use ApiPlatform\Metadata\ApiResource;
use ApiPlatform\Metadata\GraphQl\DeleteMutation;
use ApiPlatform\Metadata\GraphQl\Mutation;
use ApiPlatform\Metadata\GraphQl\Query;
use ApiPlatform\Metadata\GraphQl\QueryCollection;
use App\Resolver\BookCollectionResolver;
use App\Resolver\BookResolver;

#[ApiResource(graphQlOperations: [
    new Query(),
    new QueryCollection(),
    new Mutation(name: 'create'),
    new Mutation(name: 'update'),
    new DeleteMutation(name: 'delete'),

    new Query(name: 'retrievedQuery', resolver: BookResolver::class),
    new Query(
        name: 'notRetrievedQuery',
        resolver: BookResolver::class,
        args: []
    ),
    new Query(
        name: 'withDefaultArgsNotRetrievedQuery',
        resolver: BookResolver::class,
        read: false
    ),
    new Query(
        name: 'withCustomArgsQuery',
        resolver: BookResolver::class,
        args: [
            'id' => ['type' => 'ID!'],
            'log' => ['type' => 'Boolean!', 'description' => 'Is logging activated?'],
            'logDate' => ['type' => 'DateTime']
        ]
    ),
    new QueryCollection(name: 'collectionQuery', resolver: BookCollectionResolver::class),
])]
class Book
{
    // ...
}
```

```yaml
#The YAML syntax is only supported for Symfony
resources:
  App\Entity\Book:
    graphQlOperations:
      - class: ApiPlatform\Metadata\GraphQl\Query
      - class: ApiPlatform\Metadata\GraphQl\QueryCollection
      - class: ApiPlatform\Metadata\GraphQl\Mutation
        name: create
      - class: ApiPlatform\Metadata\GraphQl\Mutation
        name: update
      - class: ApiPlatform\Metadata\GraphQl\Mutation
        name: delete

      - class: ApiPlatform\Metadata\GraphQl\Query
        name: retrievedQuery
        resolver: App\Resolver\BookResolver
      - class: ApiPlatform\Metadata\GraphQl\Query
        name: notRetrievedQuery
        resolver: App\Resolver\BookResolver
        args: []
      - class: ApiPlatform\Metadata\GraphQl\Query
        name: withDefaultArgsNotRetrievedQuery
        resolver: App\Resolver\BookResolver
        read: false
      - class: ApiPlatform\Metadata\GraphQl\Query
        name: withCustomArgsQuery
        resolver: App\Resolver\BookResolver
        args:
          id:
            type: 'ID!'
          log:
            type: 'Boolean!'
            description: 'Is logging activated?'
          logDate:
            type: 'DateTime'
      - class: ApiPlatform\Metadata\GraphQl\QueryCollection
        name: collectionQuery
        resolver: App\Resolver\BookCollectionResolver
```

```xml
<!--The XML syntax is only supported for Symfony -->
<resources xmlns="https://api-platform.com/schema/metadata/resources-3.0"
           xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
           xsi:schemaLocation="https://api-platform.com/schema/metadata/resources-3.0
           https://api-platform.com/schema/metadata/resources-3.0.xsd">
    <resource class="App\Entity\Book">
        <graphQlOperations>
            <graphQlOperation class="ApiPlatform\Metadata\GraphQl\Query" />
            <graphQlOperation class="ApiPlatform\Metadata\GraphQl\QueryCollection" />
            <graphQlOperation class="ApiPlatform\Metadata\GraphQl\Mutation" name="create" />
            <graphQlOperation class="ApiPlatform\Metadata\GraphQl\Mutation" name="update" />
            <graphQlOperation class="ApiPlatform\Metadata\GraphQl\Mutation" name="delete" />

            <graphQlOperation class="ApiPlatform\Metadata\GraphQl\Query" name="retrievedQuery" resolver="App\Resolver\BookResolver" />
            <graphQlOperation class="ApiPlatform\Metadata\GraphQl\Query" name="notRetrievedQuery" resolver="App\Resolver\BookResolver">
                <args/>
            </graphQlOperation>
            <graphQlOperation class="ApiPlatform\Metadata\GraphQl\Query" name="withDefaultArgsNotRetrievedQuery" read="false" />
            <graphQlOperation class="ApiPlatform\Metadata\GraphQl\Query" name="withCustomArgsQuery" resolver="App\Resolver\BookResolver">
                <args>
                    <arg id="id">
                        <values>
                            <value name="type">ID!</value>
                        </values>
                    </arg>
                    <arg id="log">
                        <values>
                            <value name="type">Boolean!</value>
                            <value name="description">Is logging activated?</value>
                        </values>
                    </arg>
                    <arg id="logDate">
                        <values>
                            <value name="type">DateTime</value>
                        </values>
                    </arg>
                </args>
            </graphQlOperation>
            <graphQlOperation class="ApiPlatform\Metadata\GraphQl\QueryCollection" name="collectionQuery" provider="App\Resolver\BookCollectionResolver" />
        </graphQlOperations>
    </resource>
</resources>
```

</code-selector>

Note that you need to explicitly add the auto-generated queries and mutations if they are needed when configuring custom queries, like it's done for the [operations](#operations).

As you can see, it's possible to define your own arguments for your custom queries.
They are following the GraphQL type system.
If you don't define the `args` property, it will be the default ones (for example `id` for an item).
You can also use the `extraArgs` property if you want to add more arguments than the generated ones.

If you don't want API Platform to retrieve the item for you, disable the `read` provider.
Some other providers and processors [can be disabled](#disabling-system-providers-and-processors).
Another option would be to make sure there is no `id` argument.
This is the case for `notRetrievedQuery` (empty args).
Conversely, if you need to add custom arguments, make sure `id` is added among the arguments if you need the item to be retrieved automatically.

Note also that:

- If you have added your [own custom types](#custom-types), you can use them directly for your arguments types (it's the case here for `DateTime`).
- You can also add a custom description for your custom arguments. You can see the [field arguments documentation](https://webonyx.github.io/graphql-php/type-system/object-types/#field-arguments) for more options.

The arguments you have defined or the default ones and their value will be in `$context['args']` of your resolvers.

Your custom queries will be available like this:

```graphql
{
  retrievedQueryBook(id: "/books/56") {
    title
  }

  notRetrievedQueryBook {
    title
  }

  withDefaultArgsNotRetrievedQueryBook(id: "/books/56") {
    title
  }

  withCustomArgsQueryBook(id: "/books/23", log: true, logDate: "2019-12-20") {
    title
  }

  collectionQueryBooks {
    edges {
      node {
        title
      }
    }
  }
}
```

## Mutations

If you don't know what mutations are yet, the documentation about them is [here](https://graphql.org/learn/queries/#mutations).

For each resource, three mutations are available:

- `Mutation(name: 'create')` for creating a new resource
- `Mutation(name: 'update')` for updating an existing resource
- `DeleteMutation(name: 'delete')` for deleting an existing resource

When updating or deleting a resource, you need to pass the **IRI** of the resource as argument. See [Global Object Identifier](#global-object-identifier) for more information.

### Client Mutation ID

Following the [Relay Input Object Mutations Specification](https://github.com/facebook/relay/blob/v7.1.0/website/spec/Mutations.md#relay-input-object-mutations-specification),
you can pass a `clientMutationId` as argument and can ask its value as a field.

For example, if you delete a book:

```graphql
mutation DeleteBook($id: ID!, $clientMutationId: String!) {
  deleteBook(input: { id: $id, clientMutationId: $clientMutationId }) {
    clientMutationId
  }
}
```

### Custom Mutations

Creating custom mutations is comparable to creating [custom queries](#custom-queries).

Create your resolver:

```php
<?php
// api/src/Resolver/BookMutationResolver.php
namespace App\Resolver;

use ApiPlatform\GraphQl\Resolver\MutationResolverInterface;
use App\Entity\Book;

final class BookMutationResolver implements MutationResolverInterface
{
    /**
     * @param Book|null $item
     *
     * @return Book
     */
    public function __invoke($item, array $context)
    {
        // Mutation input arguments are in $context['args']['input'].

        // Do something with the book.
        // Or fetch the book if it has not been retrieved.

        // The returned item will pe persisted.
        return $item;
    }
}
```

As you can see, depending on how you configure your custom mutation in the resource, the item is retrieved or not.
For instance, if you don't set an `id` argument or if you disable the `read` or the `deserialize` providers (other state providers and state processors [can also be disabled](#disabling-system-providers-and-processors)),
the received item will be `null`.

Likewise, if you don't want your item to be persisted by API Platform,
you can return `null` instead of the mutated item (be careful: the response will also be `null`) or disable the `write` provider.

Don't forget the resolver is a service and you can inject the dependencies you want.

If you don't use autoconfiguration, add the tag `api_platform.graphql.mutation_resolver` to the resolver service.
If you're using Laravel, don't forget to tag the resolver service with the `ApiPlatform\GraphQl\Resolver\MutationResolverInterface`.

Note that you need to explicitly add the auto-generated queries and mutations if they are needed when configuring custom mutations, like it's done for the [operations](#operations).

As the custom queries, you can define your own arguments if you don't want to use the default ones (extracted from your resource).
The only difference with them is that, even if you define your own arguments, the `clientMutationId` will always be set.
You can also use the `extraArgs` property in case you need to add additional arguments (for instance to add the `id` argument since it is not added by default for a custom mutation).

The arguments will be in `$context['args']['input']` of your resolvers.

## Subscriptions

Subscriptions are an [RFC](https://github.com/graphql/graphql-spec/blob/master/rfcs/Subscriptions.md#rfc-graphql-subscriptions) to allow a client to receive pushed realtime data from the server.

In API Platform, the built-in subscription support is handled by using [Mercure](https://mercure.rocks/) as its underlying protocol.

### Enable Update Subscriptions for a Resource

To enable update subscriptions for a resource, these conditions have to be met:

- the [Mercure hub and bundle need to be installed and configured](mercure.md#installing-mercure-support).
- Mercure needs to be enabled for the resource.
- the `update` mutation needs to be enabled for the resource.
- the subscription needs to be enabled for the resource.

For instance, your resource should look like this:

<code-selector>

```php
<?php
// api/src/Entity/Book.php
namespace App\Entity;

use ApiPlatform\Metadata\ApiResource;
use ApiPlatform\Metadata\GraphQl\Mutation;
use ApiPlatform\Metadata\GraphQl\Subscription;

#[ApiResource(mercure: true, graphQlOperations: [
    new Mutation(name: 'update'),
    new Subscription()
])]
class Book
{
    // ...
}
```

```yaml
#The YAML syntax is only supported for Symfony
resources:
  App\Entity\Book:
    graphQlOperations:
      ApiPlatform\Metadata\GraphQl\Mutation:
        name: update
      ApiPlatform\Metadata\GraphQl\Subscription: ~
```

```xml
<!--The XML syntax is only supported for Symfony-->
<resources xmlns="https://api-platform.com/schema/metadata/resources-3.0"
           xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
           xsi:schemaLocation="https://api-platform.com/schema/metadata/resources-3.0
           https://api-platform.com/schema/metadata/resources-3.0.xsd">
    <resource class="App\Entity\Book">
        <graphQlOperations>
            <graphQlOperation class="ApiPlatform\Metadata\GraphQl\Mutation" name="update" />
            <graphQlOperation class="ApiPlatform\Metadata\GraphQl\Subscription" />
        </graphQlOperations>
    </resource>
</resources>
```

</code-selector>

### Subscribe

Doing a subscription is very similar to doing a query:

```graphql
{
  subscription {
    updateBookSubscribe(
      input: { id: "/books/1", clientSubscriptionId: "myId" }
    ) {
      book {
        title
        isbn
      }
      mercureUrl
      clientSubscriptionId
    }
  }
}
```

As you can see, you need to pass the **IRI** of the resource as argument. See [Global Object Identifier](#global-object-identifier) for more information.

You can also pass `clientSubscriptionId` as argument and can ask its value as a field.

In the payload of the subscription, the given fields of the resource will be the fields you subscribe to: if any of these fields is updated, you will be pushed their updated values.

The `mercureUrl` field is the Mercure URL you need to use to [subscribe to the updates](https://mercure.rocks/docs/getting-started#subscribing) on the client-side.

### Receiving an Update

On the client-side, you will receive the pushed updated data like you would receive the updated data if you did an `update` mutation.

For instance, you could receive a JSON payload like this:

```json
{
  "book": {
    "title": "Updated title",
    "isbn": "978-6-6344-4051-1"
  }
}
```

### Subscriptions Cache

Internally, API Platform stores the subscriptions in a cache, using the [Symfony Cache](https://symfony.com/doc/current/cache.html) if you're using the Symfony variant
or API Platform uses [Laravel cache](https://laravel.com/docs/cache) if you're using the Laravel variant.

The cache is named `api_platform.graphql.cache.subscription` and the subscription keys are generated from the subscription payload by using a SHA-256 hash.

It's recommended to use an adapter like Redis for this cache.

## Workflow of the Resolvers

API Platform resolves the queries and mutations by using its own **resolvers**.

Even if you create your [custom queries](#custom-queries) or your [custom mutations](#custom-mutations),
these resolvers will be used and yours will be called at the right time.

See the [Extending API Platform](extending.md) documentation for more information.

### Disabling system providers and processors

If you need to, you can disable some states providers and state processors, for instance if you don't want your data to be validated.

The following table lists the system states providers and states processors you can disable in your resource configuration.

| Attribute                  | Type   | Default | Description                               |
| -------------------------- | ------ | ------- | ----------------------------------------- |
| `query_parameter_validate` | `bool` | `true`  | Enables or disables `QueryParameter`      |
| `read`                     | `bool` | `true`  | Enables or disables `ReadProvider`        |
| `deserialize`              | `bool` | `true`  | Enables or disables `DeserializeProvider` |
| `validate`                 | `bool` | `true`  | Enables or disables `ValidateProcessor`   |
| `write`                    | `bool` | `true`  | Enables or disables `WriteProcessor`      |
| `serialize`                | `bool` | `true`  | Enables or disables `SerializeProcessor`  |

A provider or a processor can be disabled at the operation level:

<code-selector>

```php
<?php
// api/src/Entity/Book.php
namespace App\Entity;

use ApiPlatform\Metadata\ApiResource;
use ApiPlatform\Metadata\GraphQl\Mutation;
use ApiPlatform\Metadata\GraphQl\Query;
use ApiPlatform\Metadata\GraphQl\QueryCollection;

#[ApiResource(graphQlOperations: [
    new Query(),
    new QueryCollection(),
    new Mutation(name: 'create', write: false)
])]
class Book
{
    // ...
}
```

```yaml
#The YAML syntax is only supported for Symfony
resources:
  App\Entity\Book:
    graphQlOperations:
      ApiPlatform\Metadata\GraphQl\Query: ~
      ApiPlatform\Metadata\GraphQl\QueryCollection: ~
      ApiPlatform\Metadata\GraphQl\Mutation:
        name: create
        write: false
```

```xml
<!--The XML syntax is only supported for Symfony-->
<resources xmlns="https://api-platform.com/schema/metadata/resources-3.0"
           xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
           xsi:schemaLocation="https://api-platform.com/schema/metadata/resources-3.0
           https://api-platform.com/schema/metadata/resources-3.0.xsd">
    <resource class="App\Entity\Book">
        <graphQlOperations>
            <graphQlOperation class="ApiPlatform\Metadata\GraphQl\Query" />
            <graphQlOperation class="ApiPlatform\Metadata\GraphQl\QueryCollection" />
            <graphQlOperation class="ApiPlatform\Metadata\GraphQl\Mutation" name="create" write="false" />
        </graphQlOperations>
    </resource>
</resources>
```

</code-selector>

Or at the resource attributes level (will be also applied in REST and for all operations):

<code-selector>

```php
<?php
// api/src/Entity/Book.php
namespace App\Entity;

use ApiPlatform\Metadata\ApiResource;
use ApiPlatform\Metadata\GraphQl\Mutation;
use ApiPlatform\Metadata\GraphQl\Query;
use ApiPlatform\Metadata\GraphQl\QueryCollection;

#[ApiResource(write: false, graphQlOperations: [
    new Query(),
    new QueryCollection(),
    new Mutation(name: 'create')
])]
class Book
{
    // ...
}
```

```yaml
#The YAML syntax is only supported for Symfony

resources:
  App\Entity\Book:
    write: false
    graphQlOperations:
      ApiPlatform\Metadata\GraphQl\Query: ~
      ApiPlatform\Metadata\GraphQl\QueryCollection: ~
      ApiPlatform\Metadata\GraphQl\Mutation:
        name: create
```

```xml
<!--The XML syntax is only supported for Symfony-->
<resources xmlns="https://api-platform.com/schema/metadata/resources-3.0"
           xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
           xsi:schemaLocation="https://api-platform.com/schema/metadata/resources-3.0
           https://api-platform.com/schema/metadata/resources-3.0.xsd">
    <resource class="App\Entity\Greeting" write="false">
        <graphQlOperations>
            <graphQlOperation class="ApiPlatform\Metadata\GraphQl\Query" />
            <graphQlOperation class="ApiPlatform\Metadata\GraphQl\QueryCollection" />
            <graphQlOperation class="ApiPlatform\Metadata\GraphQl\Mutation" name="create" />
        </graphQlOperations>
    </resource>
</resources>
```

</code-selector>

## Events

No events are sent by the resolvers in API Platform. If you want to add your custom logic, [extending API Platform](extending.md) is
the recommended way to do it.

## Filters

Filters are supported out-of-the-box. Follow the [filters](filters.md) documentation and your filters will be available as arguments of queries.

However, you don't necessarily have the same needs for your GraphQL endpoint as for your REST one.

In the `QueryCollection` attribute, you can choose to decorrelate the GraphQL filters.
In order to keep the default behavior (possibility to fetch, delete, update or create), define all the auto-generated operations (`Query` ,`QueryCollection`, `DeleteMutation`, and the `update` and `create` `Mutation`).

For example, this entity will have a search filter for REST and a date filter for GraphQL:

<code-selector>

```php
<?php
// api/src/Entity/Offer.php
namespace App\Entity;

use ApiPlatform\Metadata\ApiResource;
use ApiPlatform\Metadata\GraphQl\DeleteMutation;
use ApiPlatform\Metadata\GraphQl\Mutation;
use ApiPlatform\Metadata\GraphQl\Query;
use ApiPlatform\Metadata\GraphQl\QueryCollection;

#[ApiResource(filters: ['offer.search_filter'], graphQlOperations: [
    new Query(),
    new QueryCollection(filters: ['offer.date_filter']),
    new Mutation(name: 'create'),
    new Mutation(name: 'update'),
    new DeleteMutation(name: 'delete')
])]
class Offer
{
    // ...
}
```

```yaml
#The YAML syntax is only supported for Symfony
resources:
  App\Entity\Book:
    filters: ['offer.search_filter']
    graphQlOperations:
      - class: ApiPlatform\Metadata\GraphQl\Query
      - class: ApiPlatform\Metadata\GraphQl\QueryCollection
        filters: ['offer.date_filter']
      - class: ApiPlatform\Metadata\GraphQl\Mutation
        name: create
      - class: ApiPlatform\Metadata\GraphQl\Mutation
        name: update
      - class: ApiPlatform\Metadata\GraphQl\Mutation
        name: delete
```

```xml
<!--The XML syntax is only supported for Symfony-->
<resources xmlns="https://api-platform.com/schema/metadata/resources-3.0"
           xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
           xsi:schemaLocation="https://api-platform.com/schema/metadata/resources-3.0
           https://api-platform.com/schema/metadata/resources-3.0.xsd">
    <resource class="App\Entity\Book">
        <filters>
            <filter>offer.search_filter</filter>
        </filters>

        <graphQlOperations>
            <graphQlOperation class="ApiPlatform\Metadata\GraphQl\Query" />
            <graphQlOperation class="ApiPlatform\Metadata\GraphQl\QueryCollection">
                <fitlers>
                    <filter>offer.date_filter</filter>
                </fitlers>
            </graphQlOperation>
            <graphQlOperation class="ApiPlatform\Metadata\GraphQl\Mutation" name="create" />
            <graphQlOperation class="ApiPlatform\Metadata\GraphQl\Mutation" name="update" />
            <graphQlOperation class="ApiPlatform\Metadata\GraphQl\Mutation" name="delete" />
        </graphQlOperations>
    </resource>
</resources>
```

</code-selector>

### Syntax for Filters with a List of Key / Value Arguments

Some filters like the [exists filter](doctrine-filters.md#exists-filter) or the [order filter](doctrine-filters.md#order-filter-sorting) take a list of key / value as arguments.

The first syntax coming to mind to use them is to write:

```graphql
{
  offers(order: { id: "ASC", name: "DESC" }) {
    edges {
      node {
        id
        name
      }
    }
  }
}
```

However this syntax has a problematic issue: it doesn't keep the order of the arguments.
These filters usually need a proper order to give results as expected.

That's why this syntax needs to be used instead:

```graphql
{
  offers(order: [{ id: "ASC" }, { name: "DESC" }]) {
    edges {
      node {
        id
        name
      }
    }
  }
}
```

Since a list is used for the arguments, the order is preserved.

### Filtering on Nested Properties

Unlike for REST, all built-in filters support nested properties using the underscore (`_`) syntax instead of the dot (`.`) syntax, e.g.:

```php
<?php
// api/src/Entity/Offer.php
namespace App\Entity;

use ApiPlatform\Metadata\ApiFilter;
use ApiPlatform\Metadata\ApiResource;
use ApiPlatform\Doctrine\Orm\Filter\OrderFilter;
use ApiPlatform\Doctrine\Orm\Filter\SearchFilter;

#[ApiResource]
#[ApiFilter(OrderFilter::class, properties: ['product.releaseDate'])]
#[ApiFilter(SearchFilter::class, properties: ['product.color' => 'exact'])]
class Offer
{
    // ...
}
```

The above allows you to find offers by their respective product's color like for the REST API.
You can then filter using the following syntax:

```graphql
{
  offers(product_color: "red") {
    edges {
      node {
        id
        product {
          name
          color
        }
      }
    }
  }
}
```

Or order your results like:

```graphql
{
  offers(order: [{ product_releaseDate: "DESC" }]) {
    edges {
      node {
        id
        product {
          name
          color
        }
      }
    }
  }
}
```

Another difference with the REST API filters is that the keyword `_list` must be used instead of the traditional `[]` to filter over multiple values.

For example, if you want to search the offers with a green or a red product you can use the following syntax:

```graphql
{
  offers(product_color_list: ["red", "green"]) {
    edges {
      node {
        id
        product {
          name
          color
        }
      }
    }
  }
}
```

## Pagination

API Platform natively enables a cursor-based pagination for collections.
It supports [GraphQL's Complete Connection Model](https://graphql.org/learn/pagination/#complete-connection-model) and is compatible with [GraphQL Cursor Connections Specification](https://relay.dev/graphql/connections.htm).

A page-based pagination can also be enabled per resource or per operation.

### Using the Cursor-based Pagination

Here is an example query leveraging the pagination system:

```graphql
{
  offers(first: 10, after: "cursor") {
    totalCount
    pageInfo {
      endCursor
      hasNextPage
    }
    edges {
      cursor
      node {
        id
      }
    }
  }
}
```

Two pairs of parameters work with the query:

- `first` and `after`;
- `last` and `before`.

More precisely:

- `first` corresponds to the items per page starting from the beginning;
- `after` corresponds to the `cursor` from which the items are returned.

- `last` corresponds to the items per page starting from the end;
- `before` corresponds to the `cursor` from which the items are returned, from a backwards point of view.

The current page always has a `startCursor` and an `endCursor`, present in the `pageInfo` field.

To get the next page, you would add the `endCursor` from the current page as the `after` parameter.

```graphql
{
  offers(first: 10, after: "endCursor") {
  }
}
```

For the previous page, you would add the `startCursor` from the current page as the `before` parameter.

```graphql
{
  offers(last: 10, before: "startCursor") {
  }
}
```

How do you know when you have reached the last page? It is the aim of the property `hasNextPage` or `hasPreviousPage` in `pageInfo`.
When it is false, you know it is the last page and moving forward or backward will give you an empty result.

### Using the Page-based Pagination

In order to use the page-based pagination, you need to enable it in the resource.

For instance at the operation level:

<code-selector>

```php
<?php
// api/src/Entity/Offer.php
namespace App\Entity;

use ApiPlatform\Metadata\ApiResource;
use ApiPlatform\Metadata\GraphQl\DeleteMutation;
use ApiPlatform\Metadata\GraphQl\Mutation;
use ApiPlatform\Metadata\GraphQl\Query;
use ApiPlatform\Metadata\GraphQl\QueryCollection;

#[ApiResource(graphQlOperations: [
    new Query(),
    new QueryCollection(paginationType: 'page'),
    new Mutation(name: 'create'),
    new Mutation(name: 'update'),
    new DeleteMutation(name: 'delete')
])]
class Offer
{
    // ...
}
```

```yaml
#The YAML syntax is only supported for Symfony
resources:
  App\Entity\Book:
    graphQlOperations:
      - class: ApiPlatform\Metadata\GraphQl\Query
      - class: ApiPlatform\Metadata\GraphQl\QueryCollection
        paginationType: page
      - class: ApiPlatform\Metadata\GraphQl\Mutation
        name: create
      - class: ApiPlatform\Metadata\GraphQl\Mutation
        name: update
      - class: ApiPlatform\Metadata\GraphQl\Mutation
        name: delete
```

```xml
<!--The XML syntax is only supported for Symfony-->
<resources xmlns="https://api-platform.com/schema/metadata/resources-3.0"
           xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
           xsi:schemaLocation="https://api-platform.com/schema/metadata/resources-3.0
           https://api-platform.com/schema/metadata/resources-3.0.xsd">
    <resource class="App\Entity\Book">
        <graphQlOperations>
            <graphQlOperation class="ApiPlatform\Metadata\GraphQl\Query" />
            <graphQlOperation class="ApiPlatform\Metadata\GraphQl\QueryCollection" paginationType="page" />
            <graphQlOperation class="ApiPlatform\Metadata\GraphQl\Mutation" name="create" />
            <graphQlOperation class="ApiPlatform\Metadata\GraphQl\Mutation" name="update" />
            <graphQlOperation class="ApiPlatform\Metadata\GraphQl\Mutation" name="delete" />
        </graphQlOperations>
    </resource>
</resources>
```

</code-selector>

Or if you want to do it at the resource level:

<code-selector>

```php
<?php
// api/src/Entity/Offer.php
namespace App\Entity;

use ApiPlatform\Metadata\ApiResource;

#[ApiResource(paginationType: 'page')]
class Offer
{
    // ...
}
```

```yaml
#The YAML syntax is only supported for Symfony
resources:
  App\Entity\Book:
    paginationType: page
```

```xml
<!--The XML syntax is only supported for Symfony-->
<resources xmlns="https://api-platform.com/schema/metadata/resources-3.0"
           xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
           xsi:schemaLocation="https://api-platform.com/schema/metadata/resources-3.0
           https://api-platform.com/schema/metadata/resources-3.0.xsd">
    <resource class="App\Entity\Book" paginationType="page" />
</resources>
```

</code-selector>

Once enabled, a `page` filter will be available in the collection query (its name [can be changed in the configuration](pagination.md))
and an `itemsPerPage` filter will be available too if [client-side-pagination](pagination.md#changing-the-number-of-items-per-page-for-a-specific-resource) is enabled.

A `paginationInfo` field can be queried to obtain the following information:

- `itemsPerPage`: the number of items per page. To change it, follow the [pagination documentation](pagination.md#changing-the-number-of-items-per-page).
- `lastPage`: the last page of the collection.
- `totalCount`: the total number of items in the collection.
- `hasNextPage`: does the current collection offers a next page.

The collection items data are available in the `collection` field.

An example of a query:

```graphql
{
  offers(page: 3, itemsPerPage: 15) {
    collection {
      id
    }
    paginationInfo {
      itemsPerPage
      lastPage
      totalCount
      hasNextPage
    }
  }
}
```

### Disabling the Pagination

See also the [pagination documentation](pagination.md#disabling-the-pagination).

#### Globally

The pagination can be disabled for all GraphQL resources using this configuration:

##### Disable pagination for all GraphQL resources with Symfony

```yaml
# api/config/packages/api_platform.yaml
api_platform:
  graphql:
    collection:
      pagination:
        enabled: false
```

##### Disable pagination for all GraphQL resources with Laravel

```php
<?php
// config/api-platform.php
return [
    // ....
    'graphql' => [
        'collection' => [
            'pagination' => => [
                'enabled' => false,
            ],
        ]
    ],
];
```

#### For a Specific Resource

It can also be disabled for a specific resource (REST and GraphQL):

<code-selector>

```php
<?php
// api/src/Entity/Book.php
namespace App\Entity;

use ApiPlatform\Metadata\ApiResource;

#[ApiResource(paginationEnabled: false)]
class Book
{
    // ...
}
```

```yaml
#The YAML syntax is only supported for Symfony
resources:
  App\Entity\Book:
    paginationEnabled: false
```

```xml
<!--The XML syntax is only supported for Symfony-->
<resources xmlns="https://api-platform.com/schema/metadata/resources-3.0"
           xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
           xsi:schemaLocation="https://api-platform.com/schema/metadata/resources-3.0
           https://api-platform.com/schema/metadata/resources-3.0.xsd">
    <resource class="App\Entity\Book" paginationEnabled="false" />
</resources>
```

</code-selector>

#### For a Specific Resource Collection Operation

You can also disable the pagination for a specific collection operation:

<code-selector>

```php
<?php
// api/src/Entity/Book.php
namespace App\Entity;

use ApiPlatform\Metadata\ApiResource;
use ApiPlatform\Metadata\GraphQl\QueryCollection;

#[ApiResource(graphQlOperations: [new QueryCollection(paginationEnabled: false)])]
class Book
{
    // ...
}
```

```yaml
#The YAML syntax is only supported for Symfony
resources:
  App\Entity\Book:
    graphQlOperations:
      ApiPlatform\Metadata\GraphQl\QueryCollection:
        paginationEnabled: false
```

```xml
<!--The XML syntax is only supported for Symfony-->
<resources xmlns="https://api-platform.com/schema/metadata/resources-3.0"
           xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
           xsi:schemaLocation="https://api-platform.com/schema/metadata/resources-3.0
           https://api-platform.com/schema/metadata/resources-3.0.xsd">
    <resource class="App\Entity\Book">
        <graphQlOperations>
            <graphQlOperation class="ApiPlatform\Metadata\GraphQl\QueryCollection" paginationEnabled="false" />
        </graphQlOperations>
    </resource>
</resources>
```

</code-selector>

### Partial Pagination

[Partial pagination](pagination.md#partial-pagination) is possible with GraphQL.

When enabled, backwards pagination will not be possible, and the `hasNextPage` information will be always `false`.

## Security

To add a security layer to your queries and mutations, follow the [security](security.md) documentation.

The REST security configuration and the GraphQL one are **not** correlated.

If you have only some parts differing between REST and GraphQL, you have to redefine the common parts anyway.

In the example below, we want the same security rules as we have in REST, but we also want to allow an admin to delete a book only in GraphQL.
Please note that, it's not possible to update a book in GraphQL because the `update` operation is not defined.

<code-selector>

```php
<?php
// api/src/Entity/Book.php
namespace App\Entity;

use ApiPlatform\Metadata\ApiResource;
use ApiPlatform\Metadata\Get;
use ApiPlatform\Metadata\GraphQl\DeleteMutation;
use ApiPlatform\Metadata\GraphQl\Mutation;
use ApiPlatform\Metadata\GraphQl\Query;
use ApiPlatform\Metadata\GraphQl\QueryCollection;
use ApiPlatform\Metadata\Post;

#[ApiResource(
    security: "is_granted('ROLE_USER')",
    operations: [
        new Get(security: "is_granted('ROLE_USER') and object.owner == user", securityMessage: 'Sorry, but you are not the book owner.'),
        new Post(security: "is_granted('ROLE_ADMIN')", securityMessage: 'Only admins can add books.')
    ],
    graphQlOperations: [
        new Query(security: "is_granted('ROLE_USER') and object.owner == user"),
        new QueryCollection(security: "is_granted('ROLE_ADMIN')"),
        new DeleteMutation(name: 'delete', security: "is_granted('ROLE_ADMIN')"),
        new Mutation(name: 'create', security: "is_granted('ROLE_ADMIN')")
    ]
)]
class Book
{
    // ...
}
```

```yaml
#The YAML syntax is only supported for Symfony
resources:
  App\Entity\Book:
    security: "is_granted('ROLE_USER')"
    operations:
      ApiPlatform\Metadata\Get:
        security: "is_granted('ROLE_USER') and object.owner == user"
        securityMessage: 'Sorry, but you are not the book owner.'
      ApiPlatform\Metadata\Post:
        security: "is_granted('ROLE_ADMIN')"
        securityMessage: 'Only admins can add books.'
    graphQlOperations:
      ApiPlatform\Metadata\GraphQl\Query:
        security: "is_granted('ROLE_USER') and object.owner == user"
      ApiPlatform\Metadata\GraphQl\QueryCollection:
        security: "is_granted('ROLE_ADMIN')"
      ApiPlatform\Metadata\GraphQl\DeleteMutation:
        name: delete
        security: "is_granted('ROLE_ADMIN')"
      ApiPlatform\Metadata\GraphQl\Mutation:
        name: create
        security: "is_granted('ROLE_ADMIN')"
```

```xml
<!--The XML syntax is only supported for Symfony-->
<resources xmlns="https://api-platform.com/schema/metadata/resources-3.0"
           xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
           xsi:schemaLocation="https://api-platform.com/schema/metadata/resources-3.0
           https://api-platform.com/schema/metadata/resources-3.0.xsd">
    <resource class="App\Entity\Book" security="is_granted('ROLE_USER')">
        <operations>
            <operation class="ApiPlatform\Metadata\Get" security="is_granted('ROLE_USER') and object.owner == user" securityMessage="Sorry, but you are not the book owner." />
            <operation class="ApiPlatform\Metadata\Post" security="is_granted('ROLE_ADMIN')" securityMessage="Only admins can add books." />
        </operations>
        <graphQlOperations>
            <graphQlOperation class="ApiPlatform\Metadata\GraphQl\Query" security="is_granted('ROLE_USER') and object.owner == user" />
            <graphQlOperation class="ApiPlatform\Metadata\GraphQl\QueryCollection" security="is_granted('ROLE_ADMIN')" />
            <graphQlOperation class="ApiPlatform\Metadata\GraphQl\DeleteMutation" name="delete" security="is_granted('ROLE_ADMIN')" />
            <graphQlOperation class="ApiPlatform\Metadata\GraphQl\Mutation" name="create" security="is_granted('ROLE_ADMIN')" />
        </graphQlOperations>
    </resource>
</resources>
```

</code-selector>

### Securing Properties (Including Associations)

You may want to limit access to certain resource properties with a security expression. This can be done with the `ApiProperty` `security` attribute.

Note: adding the `ApiProperty` `security` expression to a GraphQL property will automatically make the GraphQL property type nullable (if it wasn't already).
This is because `null` is returned as the property value if access is denied via the `security` expression.

In GraphQL, it's possible to expose associations - allowing nested querying.
For example, associations can be made with Doctrine ORM's `OneToMany`, `ManyToOne`, `ManyToMany`, etc.

It's important to note that the security defined on resource operations applies only to the exposed query/mutation endpoints (e.g. `Query.users`, `Mutation.updateUser`, etc.).
Resource operation security is defined via the `security` attribute for each operation defined on the resource.
This security is _not_ applied to exposed associations.

Associations can instead be secured with the `ApiProperty` `security` attribute. This provides the flexibility to have different security depending on where an association is exposed.

To prevent traversal attacks, you should ensure that any exposed associations are secured appropriately.
A traversal attack is where a user can gain unintended access to a resource by querying nested associations, gaining access to a resource that prevents direct access (via the query endpoint).
For example, a user may be denied using `Query.getUser` to get a user, but is able to access the user through an association on an object that they do have access to (e.g. `document.createdBy`).

The following example shows how associations can be secured:

<code-selector>

```php
<?php
// api/src/Entity/User.php
namespace App\Entity;

use ApiPlatform\Metadata\ApiProperty;
use ApiPlatform\Metadata\ApiResource;
use ApiPlatform\Metadata\GraphQl\Query;
use ApiPlatform\Metadata\GraphQl\QueryCollection;
use Doctrine\ORM\Mapping as ORM;

/**
 * @ORM\Entity
 */
 #[ApiResource(graphQlOperations: [
    new Query(security: 'is_granted("VIEW", object)'),
    new QueryCollection(security: 'is_granted("ROLE_ADMIN")')
 ])]
class User
{
    // ...

    /**
     * @ORM\ManyToMany(targetEntity=Document::class, mappedBy="viewers")
     */
    #[ApiProperty(security: 'is_granted("VIEW", object)')]
    private Collection $viewableDocuments;

    /**
     * @ORM\Column(type="string", length=180, unique=true)
     */
    #[ApiProperty(security: 'is_granted("ROLE_ADMIN")')]
    private string $email;
}
```

```yaml
#The YAML syntax is only supported for Symfony
resources:
  App\Entity\User:
    graphQlOperations:
      ApiPlatform\Metadata\GraphQl\Query:
        security: "is_granted('VIEW', object)"
      ApiPlatform\Metadata\GraphQl\QueryCollection:
        security: "is_granted('ROLE_ADMIN')"

properties:
  App\Entity\User:
    viewableDocuments:
      security: "is_granted('VIEW', object)"
    email:
      security: "is_granted('ROLE_ADMIN')"
```

```xml
<!--The XML syntax is only supported for Symfony-->
<!-- resources.xml -->
<resources xmlns="https://api-platform.com/schema/metadata/resources-3.0"
           xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
           xsi:schemaLocation="https://api-platform.com/schema/metadata/resources-3.0
           https://api-platform.com/schema/metadata/resources-3.0.xsd">
    <resource class="App\Entity\User">
        <graphQlOperations>
            <graphQlOperation class="ApiPlatform\Metadata\GraphQl\Query" security="is_granted('VIEW', object)" />
            <graphQlOperation class="ApiPlatform\Metadata\GraphQl\QueryCollection" security="is_granted('ROLE_ADMIN')" />
        </graphQlOperations>
    </resource>
</resources>

<!-- properties.xml -->
<properties xmlns="https://api-platform.com/schema/metadata/properties-3.0"
           xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
           xsi:schemaLocation="https://api-platform.com/schema/metadata/properties-3.0
           https://api-platform.com/schema/metadata/properties-3.0.xsd">
    <property resource="App\Entity\User" name="viewableDocuments" security="is_granted('VIEW', object)" />
    <property resource="App\Entity\User" name="email" security="is_granted('ROLE_ADMIN')" />
</properties>
```

</code-selector>

<code-selector>

```php
<?php
// api/src/Entity/Document.php
namespace App\Entity;

use ApiPlatform\Metadata\ApiProperty;
use ApiPlatform\Metadata\ApiResource;
use ApiPlatform\Metadata\GraphQl\Query;
use ApiPlatform\Metadata\GraphQl\QueryCollection;
use Doctrine\ORM\Mapping as ORM;

/**
 * @ORM\Entity
 */
 #[ApiResource(graphQlOperations: [
    new Query(security: 'is_granted("VIEW", object)'),
    new QueryCollection(security: 'is_granted("ROLE_ADMIN")')
 ])]
class Document
{
    // ...

    /**
     * @ORM\ManyToMany(targetEntity=User::class, inversedBy="viewableDocuments")
     */
    #[ApiProperty(security: 'is_granted("VIEW", object)')]
    private Collection $viewers;

    /**
     * @ORM\ManyToOne(targetEntity=User::class)
     */
    #[ApiProperty(security: 'is_granted("VIEW", object)')]
    protected ?User $createdBy = null;
}
```

```yaml
#The YAML syntax is only supported for Symfony
resources:
  App\Entity\Document:
    graphQlOperations:
      ApiPlatform\Metadata\GraphQl\Query:
        security: "is_granted('VIEW', object)"
      ApiPlatform\Metadata\GraphQl\QueryCollection:
        security: "is_granted('ROLE_ADMIN')"

properties:
  App\Entity\Document:
    viewers:
      security: "is_granted('VIEW', object)"
    createdBy:
      security: "is_granted('VIEW', object)"
```

```xml
<!--The XML syntax is only supported for Symfony-->
<!-- resources.xml -->
<resources xmlns="https://api-platform.com/schema/metadata/resources-3.0"
           xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
           xsi:schemaLocation="https://api-platform.com/schema/metadata/resources-3.0
           https://api-platform.com/schema/metadata/resources-3.0.xsd">
    <resource class="App\Entity\User">
        <graphQlOperations>
            <graphQlOperation class="ApiPlatform\Metadata\GraphQl\Query" security="is_granted('VIEW', object)" />
            <graphQlOperation class="ApiPlatform\Metadata\GraphQl\QueryCollection" security="is_granted('ROLE_ADMIN')" />
        </graphQlOperations>
    </resource>
</resources>

<!-- properties.xml -->
<properties xmlns="https://api-platform.com/schema/metadata/properties-3.0"
           xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
           xsi:schemaLocation="https://api-platform.com/schema/metadata/properties-3.0
           https://api-platform.com/schema/metadata/properties-3.0.xsd">
    <property resource="App\Entity\Document" name="viewers" security="is_granted('VIEW', object)" />
    <property resource="App\Entity\Document" name="createdBy" security="is_granted('VIEW', object)" />
</properties>
```

</code-selector>

The above example only allows admins to see the full collection of each resource (`QueryCollection`).
Users must be granted the `VIEW` attribute on a resource to be able to query it directly (`Query`) - which would use a `Voter` to make this decision.

Similar to `Query`, all associations are secured, requiring `VIEW` access on the parent object (_not_ on the association).
This means that a user with `VIEW` access to a `Document` is able to see all users who are in the `viewers` collection, as well as the `createdBy` association.
This may be a little too open, so you could instead do a role check here to only allow admins to access these fields, or check for a different attribute that could be implemented in the voter (e.g. `VIEW_CREATED_BY`.)
Alternatively, you could still expose the users, but limit the visible fields by limiting access with `ApiProperty` `security` (such as the `User::$email` property above) or with [dynamic serializer groups](serialization.md#changing-the-serialization-context-dynamically).

## Serialization Groups

You may want to restrict some resource's attributes to your GraphQL clients.

As described in the [serialization process](serialization.md) documentation, you can use serialization groups to expose only the attributes you want in queries or in mutations.

If the (de)normalization context between GraphQL and REST is different, use the `(de)normalizationContext` key to change it in each query and mutations.

Note that:

- A **query** is only using the normalization context.
- A **mutation** is using the denormalization context for its input and the normalization context for its output.

The following example shows you what can be done:

<code-selector>

```php
<?php
// api/src/Entity/Book.php
namespace App\Entity;

use ApiPlatform\Metadata\ApiResource;
use ApiPlatform\Metadata\GraphQl\Mutation;
use ApiPlatform\Metadata\GraphQl\Query;
use ApiPlatform\Metadata\GraphQl\QueryCollection;
use Symfony\Component\Serializer\Annotation\Groups;

#[ApiResource(
    normalizationContext: ['groups' => ['read']],
    denormalizationContext: ['groups' => ['write']],
    graphQlOperations: [
        new Query(normalizationContext: ['groups' => ['query']]),
        new QueryCollection(normalizationContext: ['groups' => ['query_collection']]),
        new Mutation(
            name: 'create',
            normalizationContext: ['groups' => ['query_collection']],
            denormalizationContext: ['groups' => ['mutation']]
        )
    ]
)]
class Book
{
    // ...

    #[Groups(['read', 'write', 'query', 'query_collection'])]
    public $title;

    #[Groups(['read', 'mutation', 'query'])]
    public $author;

    // ...
}
```

```yaml
#The YAML syntax is only supported for Symfony
resources:
  App\Entity\Book:
    normalizationContext:
      groups: ['read']
    denormalizationContext:
      groups: ['write']
    graphQlOperations:
      ApiPlatform\Metadata\GraphQl\Query:
        normalizationContext:
          groups: ['query']
      ApiPlatform\Metadata\GraphQl\QueryCollection:
        normalizationContext:
          groups: ['query_collection']
      ApiPlatform\Metadata\GraphQl\Mutation:
        name: create
        normalizationContext:
          groups: ['query_collection']
        denormalizationContext:
          groups: ['mutation']
```

```xml
<!--The XML syntax is only supported for Symfony-->
<resources xmlns="https://api-platform.com/schema/metadata/resources-3.0"
           xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
           xsi:schemaLocation="https://api-platform.com/schema/metadata/resources-3.0
           https://api-platform.com/schema/metadata/resources-3.0.xsd">
    <resource class="App\Entity\Book">
        <normalizationContext>
            <values>
                <value name="groups">
                    <values>
                        <value>read</value>
                    </values>
                </value>
            </values>
        </normalizationContext>
        <denormalizationContext>
            <values>
                <value name="groups">
                    <values>
                        <value>write</value>
                    </values>
                </value>
            </values>
        </denormalizationContext>
        <graphQlOperations>
            <graphQlOperation class="ApiPlatform\Metadata\GraphQl\Query">
                <normalizationContext>
                    <values>
                        <value name="groups">
                            <values>
                                <value>query</value>
                            </values>
                        </value>
                    </values>
                </normalizationContext>
            </graphQlOperation>
            <graphQlOperation class="ApiPlatform\Metadata\GraphQl\QueryCollection">
                <normalizationContext>
                    <values>
                        <value name="groups">
                            <values>
                                <value>query_collection</value>
                            </values>
                        </value>
                    </values>
                </normalizationContext>
            </graphQlOperation>
            <graphQlOperation class="ApiPlatform\Metadata\GraphQl\Mutation" name="create">
                <normalizationContext>
                    <values>
                        <value name="groups">
                            <values>
                                <value>query_collection</value>
                            </values>
                        </value>
                    </values>
                </normalizationContext>
                <denormalizationContext>
                    <values>
                        <value name="groups">
                            <values>
                                <value>mutation</value>
                            </values>
                        </value>
                    </values>
                </denormalizationContext>
            </graphQlOperation>
        </graphQlOperations>
    </resource>
</resources>
```

</code-selector>

In this case, the REST endpoint will be able to get the two attributes of the book and to modify only its title.

The GraphQL endpoint will be able to query the title and author of an item.
It will be able to query the title of the items in the collection.
It will only be able to create a book with an author.
When doing this mutation, the author of the created book will not be returned (the title will be instead).

### Different Types when Using Different Serialization Groups

When you use different serialization groups, it will create different types in your schema.

Make sure you understand the implications when doing this: having different types means breaking the cache features in some GraphQL clients (in [Apollo Client](https://www.apollographql.com/docs/react/caching/cache-configuration/#automatic-cache-updates) for example).

For instance:

- If you use a different `normalizationContext` for a mutation, a `MyResourcePayloadData` type with the restricted fields will be generated and used instead of `MyResource` (the query type).
- If you use a different `normalizationContext` for the query of an item (`Query` attribute) and for the query of a collection (`QueryCollection` attribute), two types `MyResourceItem` and `MyResourceCollection` with the restricted fields will be generated and used instead of `MyResource` (the query type).

### Embedded Relation Input (Creation of Relation in Mutation)

By default, creating a relation when using a `create` or `update` mutation is not possible.

Indeed, the mutation expects an IRI for the relation in the input, so you need to use an existing relation.

For instance if you have the following resource:

<code-selector>

```php
<?php
// api/src/Entity/Book.php
namespace App\Entity;

use ApiPlatform\Metadata\ApiResource;
use ApiPlatform\Metadata\GraphQl\Mutation;

#[ApiResource(graphQlOperations: [new Mutation(name: 'create')])]
class Book
{
    // ...

    public string $title;

    public ?Author $author;

    // ...
}
```

```yaml
#The YAML syntax is only supported for Symfony
resources:
  App\Entity\Book:
    graphQlOperations:
      ApiPlatform\Metadata\GraphQl\Mutation:
        name: create
```

```xml
<!--The XML syntax is only supported for Symfony-->
<resources xmlns="https://api-platform.com/schema/metadata/resources-3.0"
           xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
           xsi:schemaLocation="https://api-platform.com/schema/metadata/resources-3.0
           https://api-platform.com/schema/metadata/resources-3.0.xsd">
    <resource class="App\Entity\Book">
        <graphQlOperations>
            <graphQlOperation class="ApiPlatform\Metadata\GraphQl\Mutation" name="create" />
        </graphQlOperations>
    </resource>
</resources>
```

</code-selector>

Creating a book with its author will be done like this, where `/authors/32` is the IRI of an existing resource:

```graphql
{
  mutation {
    createBook(
      input: { title: "The Name of the Wind", author: "/authors/32" }
    ) {
      book {
        title
        author {
          name
        }
      }
    }
  }
}
```

In order to create an author as the same time as a book,
you need to use the denormalization context and groups on the book and the author
(see also [the dedicated part in the serialization documentation](serialization.md#denormalization):

<code-selector>

```php
<?php
// api/src/Entity/Book.php
namespace App\Entity;

use ApiPlatform\Metadata\ApiResource;
use ApiPlatform\Metadata\GraphQl\Mutation;
use Symfony\Component\Serializer\Annotation\Groups;

#[ApiResource(graphQlOperations: [new Mutation(name: 'create', denormalizationContext: ['groups' => ['book:create']])])]
class Book
{
    // ...

    #[Groups(['book:create'])]
    public string $title;

    #[Groups(['book:create'])]
    public ?Author $author;

    // ...
}
```

```yaml
#The YAML syntax is only supported for Symfony
resources:
  App\Entity\Book:
    graphQlOperations:
      ApiPlatform\Metadata\GraphQl\Mutation:
        name: create
        denormalizationContext:
          groups: ['book:create']
```

```xml
<!--The XML syntax is only supported for Symfony-->
<resources xmlns="https://api-platform.com/schema/metadata/resources-3.0"
           xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
           xsi:schemaLocation="https://api-platform.com/schema/metadata/resources-3.0
           https://api-platform.com/schema/metadata/resources-3.0.xsd">
    <resource class="App\Entity\Book">
        <graphQlOperations>
            <graphQlOperation class="ApiPlatform\Metadata\GraphQl\Mutation" name="create">
                <denormalizationContext>
                    <values>
                        <value name="groups">
                            <values>
                                <value>book:create</value>
                            </values>
                        </value>
                    </values>
                </denormalizationContext>
            </graphQlOperation>
        </graphQlOperations>
    </resource>
</resources>
```

</code-selector>

And in the author resource:

```php
<?php
// api/src/Entity/Author.php
namespace App\Entity;

use ApiPlatform\Metadata\ApiResource;
use Symfony\Component\Serializer\Annotation\Groups;

#[ApiResource]
class Author
{
    // ...

    #[Groups(['book:create'])]
    public string $name;

    // ...
}
```

In this case, creating a book with its author can now be done like this:

```graphql
{
  mutation {
    createBook(
      input: {
        title: "The Name of the Wind"
        author: { name: "Patrick Rothfuss" }
      }
    ) {
      book {
        title
        author {
          name
        }
      }
    }
  }
}
```

## Exception and Error

### Handling Exceptions and Errors (Logging, Filtering, ...)

When there are errors (GraphQL ones, or if an exception is sent), a default error handler (`api_platform.graphql.error_handler`) is called.
Its main responsibility is to apply a formatter to them.

If you need to log the errors, or if you want to filter them, you have to decorate this service.

For instance, create a class like this:

```php
<?php
// api/src/Error/ErrorHandler.php
namespace App\Error;

use ApiPlatform\GraphQl\Error\ErrorHandlerInterface;

final class ErrorHandler implements ErrorHandlerInterface
{
    private $defaultErrorHandler;

    public function __construct(ErrorHandlerInterface $defaultErrorHandler)
    {
        $this->defaultErrorHandler = $defaultErrorHandler;
    }

    public function __invoke(array $errors, callable $formatter): array
    {
        // Log or filter the errors.

        return ($this->defaultErrorHandler)($errors, $formatter);
    }
}
```

Then register the service:

#### Register the Error handler using Symfony

<code-selector>

```yaml
# api/config/services.yaml
services:
  # ...
  App\Error\ErrorHandler:
    decorates: api_platform.graphql.error_handler
```

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<!-- api/config/services.xml -->
<container xmlns="http://symfony.com/schema/dic/services"
    xmlns:xsd="http://www.w3.org/2001/XMLSchema-instance"
    xsd:schemaLocation="http://symfony.com/schema/dic/services https://symfony.com/schema/dic/services/services-1.0.xsd">

    <services>
        <service id="App\Error\ErrorHandler"
            decorates="api_platform.graphql.error_handler"
        />
    </services>
</container>
```

```php
<?php
// api/config/services.php
namespace Symfony\Component\DependencyInjection\Loader\Configurator;

use App\Error\ErrorHandler;
use App\Mailer;

return function(ContainerConfigurator $configurator) {
    $services = $configurator->services();

    $services->set(ErrorHandler::class)
        ->decorate('api_platform.graphql.error_handler');
};
```

#### Register the Error handler using Laravel

```php
<?php

namespace App\Providers;

use App\Error\ErrorHandler as ErrorHandlerDecorated;
use ApiPlatform\GraphQl\Error\ErrorHandler;
use Illuminate\Support\ServiceProvider;

class AppServiceProvider extends ServiceProvider
{
    public function register(): void
    {
        $this->app->extend(ErrorHandler::class, function (ErrorHandler $errorHandler) {
            return new ErrorHandlerDecorated($errorHandler);
        });
    }
}
```

</code-selector>

### Formatting Exceptions and Errors

By default, if an exception is sent when resolving a query or a mutation or if there are GraphQL errors, they are normalized following the [GraphQL specification](https://github.com/graphql/graphql-spec/blob/master/spec/Section%207%20--%20Response.md#errors).

It means an `errors` entry will be returned in the response, containing the following entries: `message`, `extensions`, `locations` and `path`.
For more information, please [refer to the documentation in graphql-php](https://webonyx.github.io/graphql-php/error-handling/#default-error-formatting).

In `prod` mode, the displayed message will be a generic one, excepted for a `RuntimeException` (and all exceptions inherited from it) for which it will be its actual message.
This behavior is different from what is described in the [graphql-php documentation](https://webonyx.github.io/graphql-php/error-handling).
It's because a built-in [custom exception normalizer](#custom-exception-normalizer) is used to normalize the `RuntimeException` and change the default behavior.

If you are in `dev` mode, more entries will be added in the response: `debugMessage` (containing the actual exception message, for instance in the case of a `LogicException`) and `trace` (the formatted exception trace).

For some specific exceptions, built-in [custom exception normalizers](#custom-exception-normalizer) are also used to add more information.
It's the case for a `HttpException` for which the `status` entry will be added under `extensions` and for a `ValidationException` for which `status` (by default 422) and `violations` entries will be added.

#### Custom Exception Normalizer

If you want to add more specific behaviors depending on the exception or if you want to change the behavior of the built-in ones, you can do so by creating your own normalizer.

Please follow the [Symfony documentation to create a custom normalizer](https://symfony.com/doc/current/serializer/custom_normalizer.html).

The code should look like this:

```php
<?php
// api/src/Serializer/Exception/MyExceptionNormalizer.php
namespace App\Serializer\Exception;

use App\Exception\MyException;
use GraphQL\Error\Error;
use GraphQL\Error\FormattedError;
use Symfony\Component\Serializer\Normalizer\NormalizerInterface;

final class MyExceptionNormalizer implements NormalizerInterface
{
    public function normalize($object, $format = null, array $context = []): array
    {
        $exception = $object->getPrevious();
        $error = FormattedError::createFromException($object);

        // Add your logic here and add your specific data in the $error array (in the 'extensions' entry to follow the GraphQL specification).
        // $error['extensions']['yourEntry'] = ...;

        return $error;
    }

    public function supportsNormalization($data, $format = null): bool
    {
        return $data instanceof Error && $data->getPrevious() instanceof MyException;
    }
}
```

You can see that, in the `normalize` method, you should add a call to `FormattedError::createFromException` in order to have the same behavior as the other normalizers.

When registering your custom normalizer, you can add a priority to order your normalizers between themselves.

If you use a positive priority (or no priority), your normalizer will always be called before the built-in normalizers.
For instance, you can register a custom normalizer like this:

```yaml
# api/config/services.yaml
services:
  App\Serializer\Exception\MyExceptionNormalizer:
    tags:
      - { name: 'serializer.normalizer', priority: 12 }
```

## Name Conversion

You can modify how the property names of your resources are converted into field and filter names of your GraphQL schema.

By default, the property name will be used without conversion. If you want to apply a name converter, follow the [Name Conversion documentation](serialization.md#name-conversion).

For instance, your resource can have properties in camelCase:

```php
<?php
// api/src/Entity/Book.php
namespace App\Entity;

use ApiPlatform\Metadata\ApiResource;
use ApiPlatform\Metadata\ApiFilter;
use ApiPlatform\Doctrine\Orm\Filter\SearchFilter;

#[ApiResource]
#[ApiFilter(SearchFilter::class, properties: ['publicationDate' => 'partial'])]
class Book
{
    // ...

    public $publicationDate;

    // ...
}
```

By default, with the search filter, the query to retrieve a collection will be:

```graphql
{
  books(publicationDate: "2010") {
    edges {
      node {
        publicationDate
      }
    }
  }
}
```

But if you use the `CamelCaseToSnakeCaseNameConverter`, it will be:

```graphql
{
  books(publication_date: "2010") {
    edges {
      node {
        publication_date
      }
    }
  }
}
```

### Nesting Separator

If you use snake_case, you can wonder how to make the difference between an underscore and the separator of the nested fields in the filter names, by default an underscore too.

For instance if you have this resource:

```php
<?php
// api/src/Entity/Book.php
namespace App\Entity;

use ApiPlatform\Metadata\ApiResource;
use ApiPlatform\Metadata\ApiFilter;
use ApiPlatform\Doctrine\Orm\Filter\SearchFilter;

#[ApiResource]
#[ApiFilter(SearchFilter::class, properties: ['relatedBooks.title' => 'exact'])]
class Book
{
    // ...

    public $title;

    #[ORM\OneToMany(targetEntity: Book::class)]
    public $relatedBooks;

    // ...
}
```

You would need to use the search filter like this:

```graphql
{
  books(related_books_title: "The Fitz and the Fool") {
    edges {
      node {
        title
      }
    }
  }
}
```

To avoid this issue, you can configure the nesting separator to use, for example, `__` instead of `_`:

#### Modifying nesting separator for GraphQL with Symfony

```yaml
# api/config/packages/api_platform.yaml
api_platform:
  graphql:
    nesting_separator: __
# ...
```

In this case, your query will be:

```graphql
{
  books(related_books__title: "The Fitz and the Fool") {
    edges {
      node {
        title
      }
    }
  }
}
```

#### Modifying nesting separator for GraphQL with Laravel

```php
<?php
// config/api-platform.php
return [
    // ....
    'graphql' => [
        'nesting_separator' => '__'
    ],
];
```

In this case, your query will be:

```graphql
{
  books(related_books__title: "The Fitz and the Fool") {
    edges {
      node {
        title
      }
    }
  }
}
```

Much better, isn't it?

## Custom Types

You might need to add your own types to your GraphQL application.

Create your type class by implementing the interface `ApiPlatform\GraphQl\Type\Definition\TypeInterface`.

You should extend the `GraphQL\Type\Definition\ScalarType` class too to take advantage of its useful methods.

For instance, to create a custom `DateType`:

```php
<?php
namespace App\Type\Definition;

use ApiPlatform\GraphQl\Type\Definition\TypeInterface;
use GraphQL\Error\Error;
use GraphQL\Language\AST\StringValueNode;
use GraphQL\Type\Definition\ScalarType;
use GraphQL\Utils\Utils;

final class DateTimeType extends ScalarType implements TypeInterface
{
    public function __construct()
    {
        $this->name = 'DateTime';
        $this->description = 'The `DateTime` scalar type represents time data.';

        parent::__construct();
    }

    public function getName(): string
    {
        return $this->name;
    }

    public function serialize($value)
    {
        // Already serialized.
        if (\is_string($value)) {
            return (new \DateTime($value))->format('Y-m-d');
        }

        if (!($value instanceof \DateTime)) {
            throw new Error(sprintf('Value must be an instance of DateTime to be represented by DateTime: %s', Utils::printSafe($value)));
        }

        return $value->format(\DateTime::ATOM);
    }

   public function parseValue($value)
    {
        if (!\is_string($value)) {
            throw new Error(sprintf('DateTime cannot represent non string value: %s', Utils::printSafeJson($value)));
        }

        if (false === \DateTime::createFromFormat(\DateTime::ATOM, $value)) {
            throw new Error(sprintf('DateTime cannot represent non date value: %s', Utils::printSafeJson($value)));
        }

        // Will be denormalized into a \DateTime.
        return $value;
    }

    public function parseLiteral($valueNode, ?array $variables = null)
    {
        if ($valueNode instanceof StringValueNode && false !== \DateTime::createFromFormat(\DateTime::ATOM, $valueNode->value)) {
            return $valueNode->value;
        }

        // Intentionally without message, as all information already in wrapped Exception
        throw new \Exception();
    }
}
```

You can also check the documentation of [graphql-php](https://webonyx.github.io/graphql-php/type-definitions/scalars/#writing-custom-scalar-types).

The big difference in API Platform is that the value is already serialized when it's received in your type class.
Similarly, you would not want to denormalize your parsed value since it will be done by API Platform later.

### Custom Types config for Symfony

If you use autoconfiguration (the default Symfony configuration) in your application, then you are done!

Else, you need to tag your type class like this, if you're using Symfony :

```yaml
# api/config/services.yaml
services:
  # ...
  App\Type\Definition\DateTimeType:
    tags:
      - { name: api_platform.graphql.type }
```

Your custom type is now registered and is available in the `TypesContainer`.

To use it please [modify the extracted types](#modify-the-extracted-types) or use it directly in [custom queries](#custom-queries) or [custom mutations](#custom-mutations).

### Custom Types config for Laravel

If you are using Laravel tag your type with:

```php
<?php

namespace App\Providers;

use ApiPlatform\GraphQl\Type\Definition\TypeInterface;
use App\Type\Definition\DateTimeType
use Illuminate\Support\ServiceProvider;

class AppServiceProvider extends ServiceProvider
{
    public function register(): void
    {
        $this->app->tag([DateTimeType::class], TypeInterface::class);
    }
}
```

Your custom type is now registered and is available in the `TypesContainer`.

To use it please [modify the extracted types](#modify-the-extracted-types) or use it directly in [custom queries](#custom-queries) or [custom mutations](#custom-mutations).

## Modify the Extracted Types

The GraphQL schema and its types are extracted from your resources.
In some cases, you would want to modify the extracted types for instance to use your custom ones.

To do so, you need to decorate the `api_platform.graphql.type_converter` service:

### Symfony TypeConverter Decoration

```yaml
# api/config/services.yaml
services:
  # ...
  'App\Type\TypeConverter':
    decorates: api_platform.graphql.type_converter
```

### Laravel TypeConverter Decoration

```php
<?php

namespace App\Providers;

use App\Type\TypeConverter;
use ApiPlatform\GraphQl\Type\TypeConverterInterface;
use Illuminate\Support\ServiceProvider;

class AppServiceProvider extends ServiceProvider
{
    public function register(): void
    {
        $this->app->extend(TypeConverterInterface::class, function (TypeConverterInterface $typeConverter) {
            return new TypeConverter($typeConverter);
        });
    }
}
```

Then, your class needs to look like this:

```php
<?php
namespace App\Type;

use ApiPlatform\GraphQl\Type\TypeConverterInterface;
use ApiPlatform\Metadata\GraphQl\Operation;
use App\Entity\Book;
use GraphQL\Type\Definition\Type as GraphQLType;
use Symfony\Component\PropertyInfo\Type;

final class TypeConverter implements TypeConverterInterface
{

    public function __construct(private readonly TypeConverterInterface $defaultTypeConverter) {}

    public function convertType(Type $type, bool $input, Operation $rootOperation, string $resourceClass, string $rootResource, ?string $property, int $depth)
    {
        if ('publicationDate' === $property
            && Book::class === $rootResource
        ) {
            return 'DateTime';
        }

        return $this->defaultTypeConverter->convertType($type, $input, $rootOperation, $resourceClass, $rootResource, $property, $depth);
    }

    public function resolveType(string $type): ?GraphQLType
    {
        return $this->defaultTypeConverter->resolveType($type);
    }
}
```

In this case, the `publicationDate` property of the `Book` class will have a custom `DateTime` type.

You can even apply this logic for a kind of property. Replace the previous condition with something like this:

```php
if (Type::BUILTIN_TYPE_OBJECT === $type->getBuiltinType()
    && is_a($type->getClassName(), \DateTimeInterface::class, true)
) {
    return 'DateTime';
}
```

All `DateTimeInterface` properties will have the `DateTime` type in this example.

## Changing the Serialization Context Dynamically

[As REST](serialization.md#changing-the-serialization-context-dynamically), it's possible to add dynamically a (de)serialization group when resolving a query or a mutation.

There are some differences though.

The service is `api_platform.graphql.serializer.context_builder` and the method to override is `create`.

The decorator could be like this:

### Symfony Serialization Context Decoration

```php
<?php
namespace App\Serializer;

use ApiPlatform\GraphQl\Serializer\SerializerContextBuilderInterface;
use App\Entity\Book;
use Symfony\Component\Security\Core\Authorization\AuthorizationCheckerInterface;

final class BookContextBuilder implements SerializerContextBuilderInterface
{
    public function __construct(
        private readonly SerializerContextBuilderInterface $decorated,
        private readonly AuthorizationCheckerInterface $authorizationChecker
    ) {}

    public function create(?string $resourceClass, string $operationName, array $resolverContext, bool $normalization): array
    {
        $context = $this->decorated->create($resourceClass, $operationName, $resolverContext, $normalization);
        $resourceClass = $context['resource_class'] ?? null;

        if ($resourceClass === Book::class && isset($context['groups']) && $this->authorizationChecker->isGranted('ROLE_ADMIN') && false === $normalization) {
            $context['groups'][] = 'admin:input';
        }

        return $context;
    }
}
```

### Laravel Serialization Context Decoration

```php
<?php

namespace App\Services;

use App\Models\Book;
use ApiPlatform\GraphQl\Serializer\SerializerContextBuilderInterface;
use Illuminate\Support\Facades\Auth;

final class BookContextBuilder implements SerializerContextBuilderInterface
{
    public function __construct(private readonly SerializerContextBuilderInterface $decorated) {}

    public function create(?string $resourceClass, string $operationName, array $resolverContext, bool $normalization): array
    {
        $context = $this->decorated->create($resourceClass, $operationName, $resolverContext, $normalization);
        $resourceClass = $context['resource_class'] ?? null;

        if ($resourceClass === Book::class && isset($context['groups']) && $this->isAdmin() && !$normalization) {
            $context['groups'][] = 'admin:input';
        }

        return $context;
    }

    private function isAdmin(): bool
    {
        $user = Auth::user();

        return $user && $user->role === 'admin';
    }
}
```

## Export the Schema in SDL

> [!WARNING]
> This command is not yet available with Laravel, you're welcome to contribute [on GitHub](https://github.com/api-platform/core)

You may need to export your schema in SDL (Schema Definition Language) to import it in some tools.

The `api:graphql:export` command is provided to do so:

```shell-session
    bin/console api:graphql:export -o path/to/your/volume/schema.graphql
```

Since the command prints the schema to the output if you don't use the `-o` option, you can also use this command:

```shell-session
    bin/console api:graphql:export > path/in/host/schema.graphql
```

## Handling File Upload with Symfony

If you use Symfony, please follow the [file upload documentation](../symfony/file-upload.md), only the differences
will be documented here.

The file upload with GraphQL follows the [GraphQL multipart request specification](https://github.com/jaydenseric/graphql-multipart-request-spec).

You can also upload multiple files at the same time.

### Configuring the Entity Receiving the Uploaded File

Configure the entity by adding a [custom mutation resolver](#custom-mutations):

```php
<?php
// api/src/Entity/MediaObject.php
namespace App\Entity;

use ApiPlatform\Metadata\ApiProperty;
use ApiPlatform\Metadata\ApiResource;
use ApiPlatform\Metadata\GraphQl\Mutation;
use App\Resolver\CreateMediaObjectResolver;
use Doctrine\ORM\Mapping as ORM;
use Symfony\Component\HttpFoundation\File\File;
use Symfony\Component\Serializer\Annotation\Groups;
use Symfony\Component\Validator\Constraints as Assert;
use Vich\UploaderBundle\Mapping\Annotation as Vich;

/**
 * @Vich\Uploadable
 */
#[ORM\Entity]
#[ApiResource(
    normalizationContext: ['groups' => ['media_object_read']],
    types: ['https://schema.org/MediaObject'],
    graphQlOperations: [
        new Mutation(
            name: 'upload',
            resolver: CreateMediaObjectResolver::class,
            deserialize: false,
            args: [
                'file' => [
                    'type' => 'Upload!',
                    'description' => 'The file to upload'
                ]
            ]
        )
    ]
)]
class MediaObject
{
    #[ORM\Id, ORM\Column, ORM\GeneratedValue]
    protected ?int $id = null;

    #[ApiProperty(types: ['https://schema.org/contentUrl'])]
    #[Groups(['media_object_read'])]
    public ?string $contentUrl = null;

    /**
     * @Vich\UploadableField(mapping="media_object", fileNameProperty="filePath")
     */
    #[Assert\NotNull(groups: ['media_object_create'])]
    public ?File $file = null;

    #[ORM\Column(nullable: true)]
    public ?string $filePath = null;

    public function getId(): ?int
    {
        return $this->id;
    }
}
```

As you can see, a dedicated type `Upload` is used in the argument of the `upload` mutation.
If you need to upload multiple files, replace `'file' => ['type' => 'Upload!', 'description' => 'The file to upload']`
with `'files' => ['type' => '[Upload!]!', 'description' => 'Files to upload']`.

You don't need to create it, it's provided in API Platform.

### Resolving the File Upload

The corresponding resolver you added in the resource configuration should be written like this:

```php
<?php
// api/src/Resolver/CreateMediaObjectResolver.php
namespace App\Resolver;

use ApiPlatform\GraphQl\Resolver\MutationResolverInterface;
use App\Entity\MediaObject;
use Symfony\Component\HttpFoundation\File\UploadedFile;

final class CreateMediaObjectResolver implements MutationResolverInterface
{
    /**
     * @param null $item
     */
    public function __invoke($item, array $context): MediaObject
    {
        $uploadedFile = $context['args']['input']['file'];

        $mediaObject = new MediaObject();
        $mediaObject->file = $uploadedFile;

        return $mediaObject;
    }
}
```

For handling the upload of multiple files, iterate over `$context['args']['input']['files']`.

### Normalization of MediaObjects

In the constructor of the `MediaObjectNormalizer`, the GraphQL Normalizer must be injected:  

```PHP
<?php
// api/src/Serializer/MediaObjectNormalizer.php

...

public function __construct(
    #[Autowire(service: 'api_platform.graphql.normalizer.item')]
    private readonly NormalizerInterface $normalizer,
    private readonly StorageInterface $storage
) {}

...
```

### Using the `createMediaObject` Mutation

Following the specification, the upload must be done with a `multipart/form-data` content type.

You need to enable it in the [allowed formats of API Platform](content-negotiation.md#configuring-formats-globally):

#### Modifying allowed formats with Symfony

```yaml
# api/config/packages/api_platform.yaml
api_platform:
  formats:
    # ...
    multipart: ['multipart/form-data']
```

#### Modifying allowed formats with Laravel

```php
<?php
// config/api-platform.php
return [
    // ....
    'formats' => [
        // ...
        'multipart' => ['multipart/form-data']
    ],
];
```

You can now upload files using the `createMediaObject` mutation, for details check [GraphQL multipart request specification](https://github.com/jaydenseric/graphql-multipart-request-spec)
and for an example implementation for the Apollo client check out [Apollo Upload Client](https://github.com/jaydenseric/apollo-upload-client).

```graphql
mutation CreateMediaObject($file: Upload!) {
  uploadMediaObject(input: { file: $file }) {
    mediaObject {
      id
      contentUrl
    }
  }
}
```

## Change Default Descriptions

By default, API Platform generates descriptions for mutations and subscriptions.

If you want to change them, or add some for queries, you can do it in the resource declaration, at the operation level, with the `description` attribute.

For instance, if you want to change the description of the `create` mutation:

```php
<?php
// api/src/Entity/Book.php
namespace App\Entity;

use ApiPlatform\Metadata\ApiResource;
use ApiPlatform\Metadata\GraphQl\Mutation;

#[ApiResource(graphQlOperations: [
    new Mutation(name: 'create', description: 'My custom description.')
])]
class Book
{
    // ...
}
```

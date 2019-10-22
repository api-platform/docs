# GraphQL Support

[GraphQL](https://graphql.org/) is a query language made to communicate with an API and therefore is an alternative to REST.

It has some advantages compared to REST: it solves the over-fetching or under-fetching of data, is strongly typed, and is capable of retrieving multiple and nested data in one go, but it also comes with drawbacks. For example it creates overhead depending on the request.

API Platform creates a REST API by default. But you can choose to enable GraphQL as well.

Once enabled, you have nothing to do: your schema describing your API is automatically built and your GraphQL endpoint is ready to go!

## Enabling GraphQL

To enable GraphQL and its IDE (GraphiQL and GraphQL Playground) in your API, simply require the [graphql-php](https://webonyx.github.io/graphql-php/) package using Composer and clear the cache one more time:

    $ composer req webonyx/graphql-php && bin/console cache:clear

You can now use GraphQL at the endpoint: `https://localhost:8443/graphql`.

*Note:* If you used [Symfony Flex to install API Platform](../getting-started/installation.md#using-symfony-flex-and-composer),
the GraphQL endpoint will be: `https://localhost:8443/api/graphql`.

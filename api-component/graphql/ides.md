# IDEs

## GraphiQL

If Twig is installed in your project, go to the GraphQL endpoint with your browser. You will see a nice interface provided by GraphiQL to interact with your API.

The GraphiQL IDE can also be found at `/graphql/graphiql`.

If you need to disable it, it can be done in the configuration:

```yaml
# api/config/packages/api_platform.yaml
api_platform:
    graphql:
        graphiql:
            enabled: false
# ...
```

### Add another Location for GraphiQL

If you want to add a different location besides `/graphql/graphiql`, you can do it like this:

```yaml
# app/config/routes.yaml
graphiql:
    path: /docs/graphiql
    controller: api_platform.graphql.action.graphiql
```

## GraphQL Playground

Another IDE is by default included in API Platform: GraphQL Playground.

It can be found at `/graphql/graphql_playground`.

You can disable it if you want in the configuration:

```yaml
# api/config/packages/api_platform.yaml
api_platform:
    graphql:
        graphql_playground:
            enabled: false
# ...
```

### Add another Location for GraphQL Playground

You can add a different location besides `/graphql/graphql_playground`:

```yaml
# app/config/routes.yaml
graphql_playground:
    path: /docs/graphql_playground
    controller: api_platform.graphql.action.graphql_playground
```

## Modifying or Disabling the Default IDE

When going to the GraphQL endpoint, you can choose to launch the IDE you want.

```yaml
# api/config/packages/api_platform.yaml
api_platform:
    graphql:
        # Choose between graphiql or graphql-playground
        default_ide: graphql-playground
# ...
```

You can also disable this feature by setting the configuration value to `false`.

```yaml
# api/config/packages/api_platform.yaml
api_platform:
    graphql:
        default_ide: false
# ...
```

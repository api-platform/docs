# Extending API Platform

Because it handles the complex, tedious and repetitive task of creating an API infrastructure for you, API Platform lets you focus on what matter the most for the end user: the business logic.
To do so, API Platform provides a lot of extension points you can use to hook your own code.
Those extensions points are taken into account both by the REST and [GraphQL](../graphql/index.md) subsystems.

The following tables summarizes which extension point to use depending of what you want to do:

| Extension Point                                                                                | Usage                                                                                                                                                                                                                               |
|------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| [Data Providers](../fetching-and-persisting-data/data-providers.md)                            | adapters for custom persistence layers, virtual fields, custom hydration                                                                                                                                                            |
| [Denormalizers](../serialization/index.md)                                   | post-process objects created from the payload sent in the HTTP request body                                                                                                                                                         |
| [Voters](../security/security.md#hooking-custom-permission-checks-using-voters)                | custom authorization logic                                                                                                                                                                                                          |
| [Validation constraints](../usage-and-configuration/validation.md)                             | custom validation logic                                                                                                                                                                                                             |
| [Data Persisters](data-persisters)                                                             | custom business logic and computations to trigger before, during or after persistence (ex: mail, call to an external API...)                                                                                                        |
| [Normalizers](../serialization/add-extra-data.md)                                              | customize the resource sent to the client (add fields in JSON documents, encode codes, dates...)                                                                                                                                    |
| [Filters](../pagination-filters-sorting/index.md)                                              | create filters for collections and automatically document them (OpenAPI, GraphQL, Hydra)                                                                                                                                            |
| [Serializer Context Builders](../serialization/dynamic-context.md)                             | Changing the Serialization context (e.g. groups) dynamically                                                                                                                                                                        |
| [Messenger Handlers](../fetching-and-persisting-data/messenger.md)                             | create 100% custom, RPC, async, service-oriented endpoints (should be used in place of custom controllers because the messenger integration is compatible with both REST and GraphQL, while custom controllers only work with REST) |
| [DTOs and Data Transformers](../serialization/dto.md)                                          | use a specific class to represent the input or output data structure related to an operation                                                                                                                                        |
| [Kernel Events](../usage-and-configuration/events.md)                                          | customize the HTTP request or response (REST only, other extension points must be preferred when possible)                                                                                                                          |

## Doctrine Specific Extension Points

| Extension Point                                            | Usage                                                                                              |
|------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| [Extensions](../fetching-and-persisting-data/extensions.md)                                | Access to the query builder to change the DQL query                                                |
| [Filters](../pagination-filters-sorting/index.md) | Add filters documentations (OpenAPI, GraphQL, Hydra) and automatically apply them to the DQL query |

## Leveraging the Built-in Infrastructure Using Composition 

While most API Platform's classes are marked as `final`, built-in services are straightforward to reuse and customize [using composition](https://en.wikipedia.org/wiki/Composition_over_inheritance).

For instance, if you want to send a mail after a resource has been persisted, but still want to benefit from the native Doctrine ORM [data persister](../fetching-and-persisting-data/data-persisters.md), use [the decorator design pattern](https://en.wikipedia.org/wiki/Decorator_pattern#PHP) to wrap the native data persister in your own class sending the mail.

To replace existing API Platform's services by your decorators, [check out how to decorate services](https://symfony.com/doc/current/service_container/service_decoration.html).

<p align="center" class="symfonycasts"><a href="https://symfonycasts.com/screencast/api-platform-security/service-decoration?cid=apip"><img src="../../distribution/images/symfonycasts-player.png" alt="Service Decoration screencast"><br>Watch the Service Decoration screencast</a></p>

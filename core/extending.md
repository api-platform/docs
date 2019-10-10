# Extending API Platform

Because it handles the complex, tedious and repetitive task of creating an API infrastructure for you, API Platform lets you focus on what matter the most for the end user: the business logic.
To do so, API Platform provides a lot of extension points you can use to hook your own code.
Those extensions points are taken into account both by the REST and [GraphQL](graphql.md) subsystems.

The following tables summarizes which extension point to use depending of what you want to do:

| Extension Point                                                                                | Usage                                                                                                                                                                                                                               |
|------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| [Data Providers](data-providers.md)                                                            | Adapters for custom persistence layers, virtual fields, custom hydration                                                                                                                                                            |
| [Denormalizers](serialization.md)                                                              | Post-process objects created from the payload sent in the HTTP request body                                                                                                                                                         |
| [Voters](security.md#hooking-custom-permission-checks-using-voters)                            | Custom authorization logic                                                                                                                                                                                                          |
| [Validation constraints](validation.md)                                                        | Custom validation logic                                                                                                                                                                                                             |
| [Data Persisters](data-persisters)                                                             | Custom business logic and computations to trigger before, during or after persistence (e.g. mail, call to an external API, …)                                                                                                       |
| [Normalizers](serialization.md#decorating-a-serializer-and-adding-extra-data)                  | Customize the resource sent to the client (e.g. add fields in JSON documents, encode codes, dates, …)                                                                                                                               |
| [Filters](filters.md)                                                                          | Create filters for collections and automatically document them (OpenAPI, GraphQL, Hydra)                                                                                                                                            |
| [Serializer Context Builders](serialization.md#changing-the-serialization-context-dynamically) | Changing the Serialization context (e.g. groups) dynamically                                                                                                                                                                        |
| [Messenger Handlers](messenger.md)                                                             | Create 100% custom, RPC, async, service-oriented endpoints (should be used in place of custom controllers because the messenger integration is compatible with both REST and GraphQL, while custom controllers only work with REST) |
| [DTOs and Data Transformers](dto.md)                                                           | Use a specific class to represent the input or output data structure related to an operation                                                                                                                                        |
| [Kernel Events](events.md)                                                                     | Customize the HTTP request or response (REST only, other extension points must be preferred when possible)                                                                                                                          |

## Doctrine Specific Extension Points

| Extension Point                                            | Usage                                                                                              |
|------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| [Extensions](extensions.md)                                | Access to the query builder to change the DQL query                                                |
| [Filters](filters.md#doctrine-orm-and-mongodb-odm-filters) | Add filters documentations (OpenAPI, GraphQL, Hydra) and automatically apply them to the DQL query |

## Leveraging the Built-in Infrastructure Using Composition 

While most API Platform's classes are marked as `final`, built-in services are straightforward to reuse and customize [using composition](https://en.wikipedia.org/wiki/Composition_over_inheritance).

For instance, if you want to send a mail after a resource has been persisted, but still want to benefit from the native Doctrine ORM [data persister](data-persisters.md), use [the decorator design pattern](https://en.wikipedia.org/wiki/Decorator_pattern#PHP) to wrap the native data persister in your own class sending the mail.

To replace existing API Platform's services by your decorators, [check out how to decorate services](https://symfony.com/doc/current/service_container/service_decoration.html).

<p align="center" class="symfonycasts"><a href="https://symfonycasts.com/screencast/api-platform-security/service-decoration?cid=apip"><img src="../distribution/images/symfonycasts-player.png" alt="Service Decoration screencast"><br>Watch the Service Decoration screencast</a></p>

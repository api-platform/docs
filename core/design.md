# General Design Considerations

Since you only need to describe the structure of the data to expose, API Platform is both [a "design-first" and "code-first"](https://swagger.io/blog/api-design/design-first-or-code-first-api-development/)
API framework. However, the "design-first" methodology is strongly recommended: first you design the **public shape** of
API endpoints.

To do so, you have to write a plain old PHP object representing the input and output of your endpoint. This is the class
that is [marked with the `@ApiResource` annotation](../distribution/index.md).
This class **doesn't have** to be mapped with Doctrine ORM, or any other persistence system. It must be simple (it's usually
just a data structure with no or minimal behaviors) and will be automatically converted to [Hydra](extending-jsonld-context.md),
[OpenAPI / Swagger](swagger.md) and [GraphQL](graphql.md) documentations or schemas by API Platform (there is a 1-1 mapping
between this class and those docs).

Then, it's up to the developer to feed API Platform with an hydrated instance of this API resource object by implementing
the [`DataProviderInterface`](data-providers.md). Basically, the data provider will query the persistence system (RDBMS,
document or graph DB, external API...), and must hydrate and return the POPO that has been designed as mentioned above.

When updating a state (`POST`, `PUT`, `PATCH`, `DELETE` HTTP methods), it's up to the developer to properly persist the
data provided by API Platform's resource object [hydrated by the serializer](serialization.md).
To do so, there is another interface to implement: [`DataPersisterInterface`](data-persisters.md).

This class will read the API resource object (the one marked with `@ApiResource`) and:
 
* persist it directly in the database;
* or hydrate a DTO then trigger a command;
* or populate an event store;
* or persist the data in any other useful way.

The logic of data persisters is the responsibility of application developers, and is **out of the API Platform's scope**.

For [Rapid Application Development](https://en.wikipedia.org/wiki/Rapid_application_development), convenience and prototyping,
**if and only if the class marked with `@ApiResource` is also a Doctrine entity**, the developer can use the Doctrine
ORM's data provider and persister implementations shipped with API Platform.

In this case, the public (`@ApiResource`) and internal (Doctrine entity) data model are shared. Then, API Platform will
be able to query, filter, paginate and persist automatically data.
This is approach is super-convenient and efficient, but is probably **not a good idea** for non-[CRUD](https://en.wikipedia.org/wiki/Create,_read,_update_and_delete)
and/or large systems.
Again, it's up to the developers to use, or to not use those built-in data providers/persisters depending of the business
they are dealing with. API Platform makes it easy to create custom data providers and persisters, and to implement appropriate
patterns such as [CQS](https://www.martinfowler.com/bliki/CommandQuerySeparation.html) or [CQRS](https://martinfowler.com/bliki/CQRS.html).

Last but not least, to create [Event Sourcing](https://martinfowler.com/eaaDev/EventSourcing.html)-based systems, a convenient
approach is:

* to persist data in an event store using a custom [data persister](data-persisters.md)
* to create projections in standard RDBMS (Postgres, MariaDB...) tables or views
* to map those projections with read-only Doctrine entity classes **and** to mark those classes with `@ApiResource`

You can then benefit from the built-in Doctrine filters, sorting, pagination, auto-joins, etc provided by API Platform.

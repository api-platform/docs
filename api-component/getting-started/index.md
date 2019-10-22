# Getting started

## Introduction

[API Platform](https://api-platform.com) is a powerful **full stack** framework dedicated to API-driven
projects and [hypermedia-driven REST APIs](https://en.wikipedia.org/wiki/HATEOAS).

It embraces [JSON for Linked Data (JSON-LD)](https://json-ld.org) and [Hydra Core Vocabulary](https://www.hydra-cg.com) web
standards, but also supports [HAL](http://stateless.co/hal_specification.html), [Swagger/Open API](https://www.openapis.org/), 
[GraphQL](https://graphql.org/), XML, JSON, CSV and YAML.

With API-Platform, you can build a working and fully featured CRUD API in minutes. 

Leverage the awesome features of the tool to develop complex and high-performance API-first projects!

API-Platform is made of:
* A **PHP** library (referred as _API-Platform Core_) to create fully featured APIs:
    * Model scaffolding based on [Schema.org](https://schema.org)
    * Automatic CRUD generation
    * Machine-readable documentation of the API in the Hydra and [Swagger/Open API](../documenting-specifying-your-api/swagger.md) formats,
        guessed from PHPDoc, Serializer, Validator and Doctrine ORM / MongoDB ODM metadata
    * Nice human-readable documentation built with Swagger UI (including a sandbox) and/or ReDoc
    * [Pagination](../pagination-filters-sorting/index.md)
    * A bunch of [filters](../pagination-filters-sorting/index.md)
    * [Ordering](../pagination-filters-sorting/index.md)
    * [Validation](../usage-and-configuration/validation.md) using the Symfony Validator Component (with groups support)
    * Advanced [authentication and authorization](../security/security.md) rules
    * Errors serialization (Hydra and the [RFC 7807](https://tools.ietf.org/html/rfc7807) are supported)
    * Advanced [serialization](../serialization/index.md) thanks to the Symfony Serializer Component (groups support, relation embedding, max depth...)
    * Automatic routes registration
    * Automatic entrypoint generation giving access to all resources
    * [JWT](../security/jwt.md) and [OAuth](https://oauth.net/) support
    * Files and `\DateTime` and serialization and deserialization
    * Built-in content negotiation
    * Doctrine, MongoDb, ElasticSearch support, extensible to your own persistence system
    * HTTP Push support
    * Intensively tested (unit and functional). The [`Fixtures/` directory](https://github.com/api-platform/core/tree/master/tests/Fixtures) contains a working app covering all features of the library.
    
* **JavaScript** tools to consume those APIs in a snap:
    * An admin generator powered by react-admin
    * React / React Native / VueJS / Next.js / Quasar component generators
 
* A **Docker** distribution to develop your project in a fully-featured environment:
    * A server delivering your API (basically, PHP-FPM + nginx)
    * A PostgreSQL database
    * A Mercure hub for real-time updates
    * An HTTPS proxy supporting HTTP2/Push 
    * A Varnish Cache server enabling API Platform's built-in invalidation cache mechanism
    * A development server for your admin
    * A development server for your Progressive Web App.

* A [Helm](https://helm.sh/) chart to deploy your API in any [Kubernetes](https://kubernetes.io/) cluster.

Everything is fully customizable through a powerful [event system](events.md) and strong OOP.

One more thing, before we start: as the API Platform distribution includes [the Symfony framework](https://symfony.com),
it is compatible with most [Symfony bundles](https://flex.symfony.com)
(plugins) and benefits from [the numerous extensions points](extending.md) provided by this rock-solid foundation (events, DIC...).
Adding features like custom, service-oriented, API endpoints, JWT or OAuth authentication, HTTP caching, mail sending or
asynchronous jobs to your APIs is straightforward.

## Screencasts

<p align="center" class="symfonycasts"><a href="https://symfonycasts.com/tracks/rest?cid=apip#api-platform"><img src="../../distribution/images/symfonycasts-player.png" alt="SymfonyCasts, API Platform screencasts"></a></p>

The easiest and funniest way to learn how to use API Platform is to watch [the more than 60 screencasts available on SymfonyCasts](https://symfonycasts.com/tracks/rest?cid=apip#api-platform)!

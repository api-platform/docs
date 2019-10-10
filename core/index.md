# The API Platform Core Library

API Platform Core is an easy-to-use and powerful library to create [hypermedia-driven REST APIs](https://en.wikipedia.org/wiki/HATEOAS).
It is a component of the [API Platform framework](https://api-platform.com). It can be used as a standalone or with [the Symfony
framework](https://symfony.com) (recommended).

It embraces [JSON for Linked Data (JSON-LD)](http://json-ld.org) and [Hydra Core Vocabulary](http://www.hydra-cg.com) web
standards but also supports [HAL](http://stateless.co/hal_specification.html), [Swagger/Open API](https://www.openapis.org/), XML, JSON, CSV and YAML.

Build a working and fully featured CRUD API in minutes. Leverage the awesome features of the tool to develop complex and
high-performance API-first projects.

If you are starting a new project, the easiest way to get API Platform up is to install
the [API Platform Distribution](../distribution/index.md).

![Screenshot](../distribution/images/swagger-ui-1.png)

## Features

Here is the fully featured REST API you'll get in minutes:

* [Automatic CRUD](operations.md)
* Hypermedia (JSON-LD and HAL)
* Machine-readable documentation of the API in the Hydra and [Swagger/Open API](swagger.md) formats,
  guessed from PHPDoc, Serializer, Validator and Doctrine ORM / MongoDB ODM metadata
* Nice human-readable documentation built with Swagger UI (including a sandbox) and/or ReDoc
* [Pagination](pagination.md)
* A bunch of [filters](filters.md)
* [Ordering](default-order.md)
* [Validation](validation.md) using the Symfony Validator Component (with groups support)
* Advanced [authentication and authorization](security.md) rules
* Errors serialization (Hydra and the [RFC 7807](https://tools.ietf.org/html/rfc7807) are supported)
* Advanced [serialization](serialization.md) thanks to the Symfony Serializer Component (groups support, relation embedding, max depth, …)
* Automatic routes registration
* Automatic entrypoint generation giving access to all resources
* [JWT](jwt.md) and [OAuth](https://oauth.net/) support
* Files and `\DateTime` and serialization and deserialization

Everything is fully customizable through a powerful [event system](events.md) and strong OOP.

This bundle is extensively tested (unit and functional). The [`Fixtures/` directory](https://github.com/api-platform/core/tree/master/tests/Fixtures) contains a working app covering all features of the library.

## Screencasts

<p align="center" class="symfonycasts"><a href="https://symfonycasts.com/tracks/rest?cid=apip#api-platform"><img src="../distribution/images/symfonycasts-player.png" alt="SymfonyCasts, API Platform screencasts"></a></p>

The easiest and funniest way to learn how to use API Platform is to watch [the more than 60 screencasts available on SymfonyCasts](https://symfonycasts.com/tracks/rest?cid=apip#api-platform)!

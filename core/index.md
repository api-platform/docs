# The API Platform Core Library

API Platform Core is an easy to use and powerful library to create [hypermedia-driven REST APIs](http://en.wikipedia.org/wiki/HATEOAS).
It is a component of the [API Platform framework](https://api-platform.com). It can be used standalone or with [the Symfony
framework](https://symfony.com) (recommended).

It embraces [JSON for Linked Data (JSON-LD)](http://json-ld.org) and [Hydra Core Vocabulary](http://www.hydra-cg.com) web
standards.

Build a working and fully-featured CRUD API in minutes. Leverage the awesome features of the tool to develop complex and
high performance API-first projects.

If you are starting a new project, the easiest way to get API Platform up is to install the [API Platform Standard Edition](../distribution/api.md).

[![JSON-LD enabled](http://json-ld.org/images/json-ld-logo-64.png)](http://json-ld.org)

## Features

Here is the fully-featured REST API you'll get in minutes:

* CRUD support through the API for Doctrine entities: list, `GET`, `POST`, `PUT` and `DELETE`
* Hypermedia implementing [JSON-LD](http://json-ld.org)
* Machine-readable documentation of the API in the [Hydra](http://hydra-cg.com) format, guessed from PHPDoc, Serializer,
Validator and Doctrine ORM metadata
* Human-readable Swagger-like documentation including a sandbox automatically generated thanks to the integration with
[NelmioApiDoc](https://github.com/nelmio/NelmioApiDocBundle)
* Pagination (compliant with Hydra)
* List filters (compliant with Hydra)
* Validation using the Symfony Validator Component, with groups support
* Errors serialization (compliant with Hydra)
* Custom serialization using the Symfony Serializer Component, with groups support and the possibility to embed relations
* Automatic routes registration
* Automatic entrypoint generation giving access to all resources
* `\DateTime` serialization and deserialization
* [FOSUserBundle](https://github.com/FriendsOfSymfony/FOSUserBundle) integration (user management)
* Easy installation thanks to API Platform

Everything is fully customizable through a powerful event system and strong OOP.
This bundle is extensively tested (unit and functionnal). The [`Fixtures/` directory](https://github.com/api-platform/core/tree/master/Fixtures))
contains a working app covering all features of the library.

![Screenshot of ApiBundle integrated with NelmioApiDocBundle](images/NelmioApiDocBundle.png)

## Other resources

* (english) [Discovering API Platform v2](https://dunglas.fr/2016/05/the-first-alpha-of-api-platform-2-0-is-available/)
* (english) [Create API-First Web Apps with API Platform, a PHP Framework](http://blog.runscope.com/posts/create-api-first-web-apps-with-api-platform-a-php-framework)
* (french) [Tour d'horizon des changements dans API Platform v2](https://les-tilleuls.coop/fr/blog/article/la-premiere-alpha-d-api-platform-2-0-est-disponible)
* (french) [A la découverte de API Platform (Symfony Live Paris 2015)](https://dunglas.fr/2015/04/mes-slides-du-symfony-live-2015-a-la-decouverte-de-api-platform/)
* (french) [API-first et Linked Data avec Symfony (sfPot Lille 2015)](https://les-tilleuls.coop/slides/dunglas/slides-sfPot-2015-01-15/#/)
* (french) [Behat PHP code coverage](http://www.kitpages.fr/fr/cms/204/behat-php-code-coverage)

Previous chapter: [Testing and Specifying the API](../distribution/testing.md)
Next chapter: [Getting Started](getting-started.md)

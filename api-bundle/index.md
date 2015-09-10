# API Bundle

DunglasApiBundle is an easy to use and powerful system to create [hypermedia-driven REST APIs](http://en.wikipedia.org/wiki/HATEOAS).
It is a component of the [API Platform framework](https://api-platform.com) and it can be used
as a standalone bundle for [the Symfony framework](https://symfony.com).

It embraces [JSON for Linked Data (JSON-LD)](http://json-ld.org) and [Hydra Core Vocabulary](http://www.hydra-cg.com) web standards. 

Build a working and fully-featured CRUD API in minutes. Leverage the awesome features of the tool to develop complex and
high performance API-first projects.

[![JSON-LD enabled](http://json-ld.org/images/json-ld-logo-64.png)](http://json-ld.org)

## Features

Here is the fully-featured REST API you'll get in minutes, I promise:

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
This bundle is documented and tested with Behat (take a look at [the `features/` directory](features/)).

![Screenshot of ApiBundle integrated with NelmioApiDocBundle](images/NelmioApiDocBundle.png)

## Official documentation

1. [Getting Started](getting-started.md)
  1. [Installing DunglasApiBundle](getting-started.md#installing-dunglasapibundle)
  2. [Configuring the API](getting-started.md#configuring-the-api)
  3. [Mapping the entities](getting-started.md#mapping-the-entities)
  4. [Registering the services](getting-started.md#registering-the-services)
2. [NelmioApiDocBundle integration](nelmio-api-doc.md)
3. [Operations](operations.md)
  1. [Disabling operations](operations.md#disabling-operations)
  2. [Creating custom operations](operations.md#creating-custom-operations)
4. [Data providers](data-providers.md)
  1. [Creating a custom data provider](data-providers.md#creating-a-custom-data-provider)
  2. [Returning a paged collection](data-providers.md#returning-a-paged-collection)
  3. [Supporting filters](data-providers.md#supporting-filters)
  4. [Extending the Doctrine Data Provider](data-providers.md#extending-the-doctrine-data-provider)
5. [Filters](filters.md)
  1. [Search filter](filters.md#search-filter)
  2. [Date filter](filters.md#date-filter)
    1. [Managing `null` values](filters.md#managing-null-values)
  3. [Order filter](filters.md#order-filter)
    1. [Using a custom order query parameter name](filters.md#using-a-custom-order-query-parameter-name)
  4. [Enabling a filter for all properties of a resource](filters.md#enabling-a-filter-for-all-properties-of-a-resource)
  5. [Creating custom filters](filters.md#creating-custom-filters)
    1. [Creating custom Doctrine ORM filters](filters.md#creating-custom-doctrine-orm-filters)
    2. [Overriding extraction of properties from the request](filters.md#overriding-extraction-of-properties-from-the-request)
6. [Serialization groups and relations](serialization-groups-and-relations.md)
  1. [Using serialization groups](serialization-groups-and-relations.md#using-serialization-groups)
  2. [Annotations](serialization-groups-and-relations.md#annotations)
  3. [Embedding relations](serialization-groups-and-relations.md#embedding-relations)
    1. [Normalization](serialization-groups-and-relations.md#normalization)
    2. [Denormalization](serialization-groups-and-relations.md#denormalization)
  4. [Name conversion](serialization-groups-and-relations.md#name-conversion)
  5. [Entity identifier case](serialization-groups-and-relations.md#entity-identifier-case)
7. [Validation](validation.md)
  1. [Using validation groups](validation.md#using-validation-groups)
8. [The event system](the-event-system.md)
  1. [Retrieving list](the-event-system.md#retrieving-list)
  2. [Retrieving item](the-event-system.md#retrieving-item)
  3. [Creating item](the-event-system.md#creating-item)
  4. [Updating item](the-event-system.md#updating-item)
  5. [Deleting item](the-event-system.md#deleting-item)
  6. [JSON-LD context builder](the-event-system.md#json-ld-context-builder)
  7. [Registering an event listener](the-event-system.md#registering-an-event-listener)
9. [Resources](resources.md)
  1. [Using a custom `Resource` class](resources.md#using-a-custom-resource-class)
10. [Controllers](controllers.md)
  1. [Using a custom controller](controllers.md#using-a-custom-controller)
11. [FOSUserBundle integration](fosuser-bundle.md#fosuser-bundle-integration)
  1. [Creating a `User` entity with serialization groups](fosuser-bundle.md#creating-a-user-entity-with-serialization-groups)
12. [Using external (JSON-LD) vocabularies](external-vocabularies.md)
13. [Performances](performances.md)
  1. [Enabling the metadata cache](performances.md#enabling-the-metadata-cache)
14. [AngularJS integration](angular-integration.md)

## Other resources

### Filters

* [LoopBackApiBundle](https://github.com/theofidry/LoopBackApiBundle): provides a set of Doctrine ORM filters for more advanced query operations

### Other resources

* (english) [Create API-First Web Apps with API Platform, a PHP Framework](http://blog.runscope.com/posts/create-api-first-web-apps-with-api-platform-a-php-framework)
* (french) [A la découverte de API Platform (Symfony Live Paris 2015)](https://dunglas.fr/2015/04/mes-slides-du-symfony-live-2015-a-la-decouverte-de-api-platform/)
* (french) [API-first et Linked Data avec Symfony (sfPot Lille 2015)](https://les-tilleuls.coop/slides/dunglas/slides-sfPot-2015-01-15/#/)
* (french) [Behat PHP code coverage](http://www.kitpages.fr/fr/cms/204/behat-php-code-coverage)


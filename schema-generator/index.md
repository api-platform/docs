# The schema generator

`schema` is a command line tool part of [the API Platform framework](https://api-platform.com) that instantly generates
a PHP data model from the [Schema.org](http://schema.org) vocabulary.
Browse Schema.org, choose the types and properties you need, run our code generator and you're done! You get
a fully featured PHP data model including:

* A set of PHP entities with properties, constants (enum values), getters, setters, adders and removers. The class
hierarchy provided by Schema.org will be translated to a PHP class hierarchy with parents as `abstract` classes. The generated
code complies with [PSR](http://www.php-fig.org/) coding standards.
* Full high-quality PHPDoc for classes, properties, constants and methods extracted from Schema.org.
* Doctrine ORM annotation mapping including database columns with type guessing, relations with cardinality guessing, class
inheritance (through the `@AbstractSuperclass` annotation).
* Data validation through [Symfony Validator](http://symfony.com/doc/current/book/validation.html) annotations including
data type validation, enum support (choices) and check for required properties.
* Interfaces and [Doctrine `ResolveTargetEntityListener`](http://doctrine-orm.readthedocs.org/en/latest/cookbook/resolve-target-entity-listener.html)
support.
* Custom PHP namespace support.
* List of values provided by Schema.org with [PHP Enum](https://github.com/myclabs/php-enum) classes.

Bonus:

* The code generator is fully configurable and extendable: all features can be deactivated (e.g.: the Doctrine mapping generator)
and custom generator can be added (e.g.: a Doctrine ODM mapping generator).
* The generated code can be used as is in a [Symfony](http://symfony.com) app (but it will work too in a raw PHP project
or any other framework including [Laravel](http://laravel.com) and [Zend Framework](http://framework.zend.com/)).

## What is Schema.org?

Schema.org is a vocabulary representing common data structures and their relations. Schema.org can be exposed as [JSON-LD](http://en.wikipedia.org/wiki/JSON-LD),
[microdata](https://en.wikipedia.org/wiki/Microdata_(HTML)) and [RDFa](http://en.wikipedia.org/wiki/RDFa).
Extracting semantical data exposed in the Schema.org vocabulary is supported by a growing number of companies including
Google (Search, Gmail), Yahoo!, Bing and Yandex.

## Why use Schema.org data to generate a PHP model?

### Don't Reinvent The Wheel

Data models provided by Schema.org are popular and were proven efficient. They cover a broad spectrum of topics including
creative works, e-commerce, events, medicine, social networking, people, postal addresses, organization data, places or reviews.
Schema.org has its root in [a ton of preexisting well designed vocabularies](http://schema.rdfs.org/mappings.html) and is
successfully used by more and more websites and applications.

Pick schemas applicable to your application, generate your PHP model, then customize and specialize it to fit your needs.

### Improve SEO and user experience

Adding Schema.org markup to websites and apps increase their ranking in search engines results and enable awesome features
such as [Google Rich Snippets](https://support.google.com/webmasters/answer/99170?hl=en) and [Gmail markup](https://developers.google.com/gmail/markup/overview).

Mapping your app data model to Schema.org structures can be tedious. When using the generator, your data model will be
derived from Schema.org. Adding microdata markup to your templates or serializing your data as JSON-LD will not require
specific mapping nor adaptation. It's a matter of minutes.

### Be ready for the future

Schema.org improves the interoperability of your applications. Used with hypermedia technologies such as [Hydra](http://www.hydra-cg.com/)
it's a big step towards the semantic and machine readable web.
It opens the way to generic web API clients able to extract and process data from any website or app using such technologies.

## Documentation

* [Getting Started](getting-started.md)
* [Configuration](configuration.md)

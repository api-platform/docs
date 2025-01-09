# Getting Started

## Installation

If you use [the API Platform distribution](../distribution/index.md), the Schema Generator is already installed as a development
dependency of your project and can be invoked through Docker:

```console
docker-compose exec php \
    vendor/bin/schema
```

The Schema Generator can also [be downloaded independently as a PHAR](https://github.com/api-platform/schema-generator/releases) or installed in an existing project using [Composer](https://getcomposer.org):

```console
composer require --dev api-platform/schema-generator
```

## Model Scaffolding

Start by browsing [Schema.org](https://schema.org) (or any other RDF vocabulary) and pick types applicable to your application. Schema.org provides
tons of schemas including (but not limited to) representations of people, organizations, events, postal addresses, creative
work and e-commerce structures. Many other open vocabularies can be found on [the LOV website](https://lov.linkeddata.es/).

Then, write a simple YAML config file similar to the following.

Here we will generate a data model for an address book with the following data:

* a [`Person`](https://schema.org/Person) which inherits from [`Thing`](https://schema.org/Thing)
* a [`PostalAddress`](https://schema.org/PostalAddress) which inherits from [`ContactPoint`](https://schema.org/ContactPoint), which itself inherits from [`StructuredValue`](https://schema.org/StructuredValue), etc.

```yaml
# config/schema.yaml
# The list of types and properties we want to use
types:
    # Parent class of Person
    Thing:
        properties:
            name: ~
    Person:
        properties:
            familyName: ~
            givenName: ~
            additionalName: ~
            address: ~
    PostalAddress:
        # Disable the generation of the class hierarchy for this type
        parent: false
        properties:
            # Force the type of the addressCountry property to text
            addressCountry: { range: "Text" }
            addressLocality: ~
            addressRegion: ~
            postOfficeBoxNumber: ~
            postalCode: ~
            streetAddress: ~
```

Run the generator with this config file as parameter:

```console
vendor/bin/schema generate api/src/ api/config/schema.yaml
```

Using [the API Platform Distribution](../distribution/index.md):

```console
docker-compose exec php \
    vendor/bin/schema generate src/ config/schema.yaml
```

The corresponding PHP classes will be automatically generated in the `src/` directory!
Note that the generator takes care of creating directories corresponding to the namespace structure.

Without configuration file, the tool will build the entire Schema.org vocabulary. If no properties are specified for a given
type, all its properties will be generated.

The generator also supports enumeration generation. For subclasses of [`Enumeration`](https://schema.org/Enumeration), the
generator will automatically create a class extending the Enum type provided by [myclabs/php-enum](https://github.com/myclabs/php-enum).
Don't forget to install this library in your project. Refer you to PHP Enum documentation to see how to use it. The Symfony
validation annotation generator automatically takes care of enumerations to validate choices values.

A config file generating an enum class:

```yaml
types:
    OfferItemCondition: # The generator will automatically guess that OfferItemCondition is subclass of Enum
      properties: {} # Remove all properties of the parent class
```

### Going Further

Browse [the configuration documentation](configuration.md)

## Cardinality Extraction

The Cardinality Extractor is a standalone tool (also used internally by the generator) extracting a property's cardinality.
It extracts cardinality described with the [Web Ontology Language (OWL)](https://en.wikipedia.org/wiki/Web_Ontology_Language) vocabulary or in [GoodRelations](http://www.heppnetz.de/projects/goodrelations/). When cardinality cannot be automatically extracted, its value is set to `unknown`.

Usage:

```console
vendor/bin/schema extract-cardinalities
```

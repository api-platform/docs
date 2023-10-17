# Getting Started

## Installation

If you use [the API Platform distribution](../distribution/index.md), the Schema Generator is already installed as a development
dependency of your project and can be invoked through Docker:

```console
docker compose exec php \
    vendor/bin/schema
```

The Schema Generator can also [be downloaded independently as a PHAR](https://github.com/api-platform/schema-generator/releases) or installed in an existing project using [Composer](https://getcomposer.org):

```console
composer require --dev api-platform/schema-generator
```

## Configuration

The Schema Generator can either be used with Schema.org types (see [model scaffolding](#model-scaffolding)) or with an OpenAPI documentation (see [OpenAPI generation](#openapi-generation)).

Choose your preferred way of designing your API, and [run the generator](#usage)!

### Model Scaffolding

Start by browsing [Schema.org](https://schema.org) (or any other RDF vocabulary) and pick types applicable to your application.
Schema.org provides tons of schemas including (but not limited to) representations of people, organizations, events, postal addresses,
creative work and e-commerce structures.
Many other open vocabularies can be found on [the LOV website](https://lov.linkeddata.es/).

Then, write a simple YAML config file similar to the following.

Here we will generate a data model for an address book with the following data:

* a [`Person`](https://schema.org/Person) which inherits from [`Thing`](https://schema.org/Thing);
* a [`PostalAddress`](https://schema.org/PostalAddress) (without its class hierarchy).

```yaml
# api/config/schema.yaml
# The list of types and properties we want to use
types:
    # Parent class of Person
    Thing:
        properties:
            name: ~
    Person:
        # Enable the generation of the class hierarchy (not enabled by default)
        parent: ~
        properties:
            familyName: ~
            givenName: ~
            additionalName: ~
            address: ~
    PostalAddress:
        properties:
            # Force the type of the addressCountry property to text
            addressCountry: { range: "Text" }
            addressLocality: ~
            addressRegion: ~
            postOfficeBoxNumber: ~
            postalCode: ~
            streetAddress: ~
```

**Note:** If no properties are specified for a given type, all its properties will be generated.

The generator also supports enumeration generation. For subclasses of [`Enumeration`](https://schema.org/Enumeration), the
generator will automatically create a class extending the Enum type provided by [myclabs/php-enum](https://github.com/myclabs/php-enum).
Don't forget to install this library in your project. Refer you to PHP Enum documentation to see how to use it.
The Symfony validation annotation generator automatically takes care of enumerations to validate choices values.

A config file generating an enum class:

```yaml
types:
    OfferItemCondition: # The generator will automatically guess that OfferItemCondition is subclass of Enum
        properties: {} # Remove all properties of the parent class
```

### OpenAPI Generation

Design your API with tools like [Stoplight](https://stoplight.io/).

Export your OpenAPI documentation to a JSON or to a YAML file and place it somewhere in your project
(for instance in `api/openapi.yaml`).

Write the following config file:

```yaml
# api/config/schema.yaml
openApi:
    file: '../openapi.yaml'
```

## Usage

Run the generator with the config file as parameter:

```console
vendor/bin/schema generate api/src/ api/config/schema.yaml -vv
```

Using [the API Platform Distribution](../distribution/index.md):

```console
docker compose exec php \
    vendor/bin/schema generate src/ config/schema.yaml -vv
```

The corresponding PHP classes will be automatically generated in the `src/` directory!
Note that the generator takes care of creating directories corresponding to the namespace structure.

Without configuration file, the tool will build the entire Schema.org vocabulary.

## Load Previously Generated Files

If you launch the schema generator again, the previously generated files will be loaded.

It will try to keep as much user-added changes as possible while adding the new changes from the configuration file.

You can also choose to overwrite the file instead.

## Going Further

Browse [the configuration documentation](configuration.md).

### Cardinality Extraction

The Cardinality Extractor is a standalone tool (also used internally by the generator) extracting a property's cardinality.
It extracts cardinality described with the [Web Ontology Language (OWL)](https://en.wikipedia.org/wiki/Web_Ontology_Language) vocabulary
or in [GoodRelations](https://www.heppnetz.de/projects/goodrelations/).
When cardinality cannot be automatically extracted, its value is set to `unknown`.

Usage:

```console
vendor/bin/schema extract-cardinalities
```

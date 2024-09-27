# Configuration

The following options can be used in the configuration file.

## Customizing PHP Namespaces

Namespaces of generated PHP classes can be set globally, respectively for entities, enumerations and interfaces
(if used with [Doctrine Resolve Target Entity Listener option](#interfaces-and-doctrine-resolve-target-entity-listener)).

Example:

```yaml
namespaces:
    entity: "App\ECommerce\Entity"
    enum: "App\ECommerce\Enum"
    interface: "App\ECommerce\Model"
```

Namespaces can also be specified for a specific type. It will take precedence over any globally configured namespace.

Example:

```yaml
types:
    Thing:
        namespaces:
            class: "App\Common\Entity" # Namespace for the Thing entity (works for enumerations too)
            interface: "App\Schema\Model" # Namespace of the related interface
```

## Forcing a Field Type (Range)

RDF allows a property to have several types (ranges). However, the generator allows only one type per property.
If not configured, it will use the first defined type.
The `range` option is useful to set the type of a given property.
It can also be used to force a type (even if not in the RDF vocabulary definition).

Example:

```yaml
types:
    Brand:
        properties:
            logo: { range: "ImageObject" } # Force the range of the logo property to ImageObject (can also be a URL according to Schema.org)

    PostalAddress:
        properties:
            addressCountry: { range: "Text" } # Force the type to Text instead of Country. It will be converted to the PHP string type.
```

## Forcing a Field Cardinality

The cardinality of a property is automatically guessed.
The `cardinality` option allows to override the guessed value.
Supported cardinalities are:

* `(0..1)`: scalar, not required
* `(0..*)`: array, not required
* `(1..1)`: scalar, required
* `(1..*)`: array, required
* `(*..0)`
* `(*..1)`
* `(*..*)`

Cardinalities are enforced by the class generator, the Doctrine ORM generator and the Symfony validation generator.

Example:

```yaml
types:
    Product:
        properties:
            sku:
                cardinality: "(0..1)"
```

## Changing the Default Cardinality

When a cardinality has not been guessed, a default cardinality will be used instead.

By default, the cardinality `(1..1)` is used, but you can change it like this:

```yaml
relations:
    defaultCardinality: "(1..*)"
```

## Adding a Custom Attribute or Modifying a Generated Attribute

You can add any custom attribute you want, or you can modify the arguments of any generated attribute,
for a property, a class or even a whole vocabulary!

For instance, if you want to change the join table name and add security for a specific relation:

```yaml
types:
    Organization:
        properties:
            contactPoint:
                attributes:
                    ORM\JoinTable: { name: organization_contactPoint } # Instead of organization_contact_point by default
                    ApiProperty: { security: "is_granted('ROLE_ADMIN')" }
```

To add a custom attribute, you also need to add it in the `uses` option:

```yaml
uses:
    App\Attributes\MyAttribute: ~

types:
    Book:
        attributes:
            - ApiResource: { routePrefix: '/library' } # Add a route prefix for this resource
            - MyAttribute: ~
            # Note the optional usage of a hyphen list: it allows to preserve the order of attributes
```

## Forcing (or Enabling) a Class Parent

Override the guessed class hierarchy of a given type with this option.

Example:

```yaml
types:
    ImageObject:
        parent: Thing # Force the parent to be Thing instead of CreativeWork > MediaObject
        properties: ~
    Drug:
        parent: ~ # Enable the class hierarchy for this type
```

## Forcing a Class to be Abstract

Force a class to be (or to not be) `abstract`.
By default, it will be guessed, depending on the class hierarchy and if the class is used in a relation.

Example:

```yaml
types:
    Person:
        abstract: true
```

## Define API Platform Operations

API Platform operations can be added this way:

```yaml
types:
    Person:
        operations:
            Get: ~
            GetCollection:
                routeName: get_person_collection
```

## Forcing a Nullable Property

Force a property to be (or to not be) `nullable`.

By default, this option is `null`: the cardinality will be used to determine the nullability.
If no cardinality is found, it will be `true`.

Example:

```yaml
    Person:
        properties:
            name: { nullable: false }
```

The `#[Assert\NotNull]` constraint is automatically added.

```php
<?php

/**
 * The name of the item.
 */
#[ORM\Column]
#[Assert\NotNull]
private string $name;
```

## Forcing a Unique Property

Force a property to be (or to not be) `unique`.

By default, this option is `false`.

Example:

```yaml
    Person:
        properties:
            email: { unique: true }
```

Output:

```php
<?php
// api/src/Entity/Person.php
namespace App\Entity;

use Doctrine\ORM\Mapping as ORM;
use Symfony\Component\Validator\Constraints as Assert;
use Symfony\Bridge\Doctrine\Validator\Constraints\UniqueEntity;
use Doctrine\ORM\Mapping as ORM;

/**
 * A person (alive, dead, undead, or fictional).
 *
 * @see https://schema.org/Person
 */
#[ORM\Entity]
#[ApiResource(types: ['https://schema.org/Person'])]
#[UniqueEntity('email')]
class Person
{
    /**
     * Email address.
     *
     * @see https://schema.org/email
     */
    #[ORM\Column]
    #[Assert\Email]
    private string $email;

    // ...
}
```

## Making a Property Read-Only

A property can be marked read-only with the following configuration:

```yaml
    Person:
        properties:
            email: { writable: false }
```

In such case, no mutator method will be generated.

## Making a Property Write-Only

A property can be marked write-only with the following configuration:

```yaml
    Person:
        properties:
            email: { readable: false }
```

In this case, no getter method will be generated.

## Forcing an Embeddable Class to be Embedded

Force an `embeddable` class to be `embedded`.

Example:

```yaml
    QuantitativeValue:
        embeddable: true
    Product:
        properties:
            weight: { range: "QuantitativeValue", embedded: true }
```

Output:

```php
<?php
// api/src/Entity/Product.php
namespace App\Entity;

use ApiPlatform\Metadata\ApiProperty;
use ApiPlatform\Metadata\ApiResource;
use Doctrine\ORM\Mapping as ORM;

/**
 * Any offered product or service.
 *
 * @see https://schema.org/Product Documentation on Schema.org
 */
#[ORM\Entity]
#[ApiResource(types: ['https://schema.org/Product'])]
#[UniqueEntity('gtin13s')]
class Product
{
    /**
     * The weight of the product or person.
     *
     * @see https://schema.org/weight
     */
    #[ORM\Embedded(class: QuantitativeValue::class)]
    #[ApiProperty(iri: 'https://schema.org/weight')]
    private ?QuantitativeValue $weight = null;

    // ...
}
```

## Skipping Accessor Method Generation

It's possible to skip the generation of accessor methods. This is particularly useful combined with the `visibility: public`
option.

To skip the generation of accessor methods, use the following config:

```yaml
accessorMethods: false
```

## Using Fluent Mutator Methods

If you want to generate fluent mutator methods, like this:

```php
public function setName(?string $name): self
{
    $this->name = $name;

    return $this;
}
```

Use the following config:

```yaml
fluentMutatorMethods: true
```

## Disabling the `id` Generator

By default, the generator adds a property called `id` not provided by Schema.org.
This is useful when generating an entity for use with an ORM or an ODM but not when generating DTOs.
This behavior can be disabled with the following setting:

```yaml
id:
    generate: false
```

## Generating UUIDs

It's also possible to let the DBMS generate [UUIDs](https://en.wikipedia.org/wiki/Universally_unique_identifier) instead of auto-incremented integers:

```yaml
id:
    generationStrategy: uuid
```

## User-submitted UUIDs

To manually set a UUID instead of letting the DBMS generate it, use the following config:

```yaml
id:
    generationStrategy: uuid
    writable: true
```

## Generating Custom IDs

With this configuration option, an `$id` property of type `string` and the corresponding getters and setters will be
generated, but the DBMS will not generate anything. The ID must be set manually.

```yaml
id:
    generationStrategy: none
```

## Disabling Usage of Doctrine Collections

By default, the generator uses classes provided by the [Doctrine Collections](https://github.com/doctrine/collections) library
to store collections of entities. This is useful (and required) when using Doctrine ORM or Doctrine MongoDB ODM.
This behavior can be disabled (to fall back to standard arrays) with the following setting:

```yaml
doctrine:
    useCollection: false
```

## Changing the Field Visibility

Generated fields have a `private` visibility and are exposed through getters and setters.
The default visibility can be changed with the `fieldVisibility` option.

Example:

```yaml
fieldVisibility: "protected"
```

## Generating `Assert\Type` Attributes

It's possible to automatically generate Symfony validator's `#[Assert\Type]` attributes using the following config:

```yaml
validator:
    assertType: true
```

## Forcing Doctrine Inheritance Mapping Attribute

The generator is able to handle inheritance in a smart way:

* If a class has children and is referenced by a relation,
it will generate an inheritance mapping strategy with `#[InheritanceType]` (configurable, see below), `#[DiscriminatorColumn]` (`#[DiscriminatorField]` for ODM) and `#[DiscriminatorMap]`.
The discriminator map will be filled with all possible values.
* If a class has children but is not referenced by a relation,
it will generate a mapped superclass (`#[MappedSuperclass]`).
If this mapped superclass defines relations and is used by multiple children,
the generator will add `#[AssociationOverride]` attributes to them
(see the [related Doctrine documentation](https://www.doctrine-project.org/projects/doctrine-orm/en/current/reference/inheritance-mapping.html#association-override)),
thanks to the special `DoctrineOrmAssociationOverrideAttributeGenerator`.
* If a class has no child, an `#[Entity]` (or `#[Document]` for ODM) attribute is used.

If this behaviour does not suit you, the inheritance attribute can be forced in the following way:

```yaml
doctrine:
    inheritanceType: SINGLE_TABLE # Default: JOINED
    inheritanceAttributes:
        CustomInheritanceAttribute: []
```

## Interfaces and Doctrine Resolve Target Entity Listener

[`ResolveTargetEntityListener`](https://www.doctrine-project.org/projects/doctrine-orm/en/current/cookbook/resolve-target-entity-listener.html)
is a feature of Doctrine to keep modules independent.
It allows to specify interfaces and `abstract` classes in relation mappings.

If you set the option `useInterface` to true, the generator will generate an interface corresponding to each generated
entity and will use them in relation mappings.

To let the schema generator generate the mapping file usable with Symfony, add the following to your config file:

```yaml
doctrine:
    resolveTargetEntityConfigPath: path/to/doctrine.xml
```

The default mapping file format is XML, but you can change it to YAML with the following option:
```yaml
doctrine:
    resolveTargetEntityConfigPath: path/to/doctrine.yaml
    resolveTargetEntityConfigType: YAML # Supports XML & YAML
```

### Doctrine Resolve Target Entity Config Type

By default, the mapping file is in XML. If you want to have a YAML file, add the following:

```yaml
doctrine:
    resolveTargetEntityConfigPath: path/to/doctrine.yaml
    resolveTargetEntityConfigType: yaml
```

## Custom Schemas

The generator can use your own schema definitions.
They must be written in RDF/XML and follow the format of the [Schema.org's definition](https://schema.org/version/latest/schemaorg-current-https.rdf).
This is useful to document your [Schema.org extensions](https://schema.org/docs/extension.html) and use them
to generate the PHP data model of your application.

Example:

```yaml
vocabularies:
    - https://github.com/schemaorg/schemaorg/raw/main/data/releases/13.0/schemaorg-current-https.rdf
    - http://example.com/data/myschema.rdf # Additional types
```

You can also use any other vocabulary.
Check the [Linked Open Vocabularies](https://lov.linkeddata.es/dataset/lov/) to find one fitting your needs.

For instance, to generate a data model from the [Video Game Ontology](http://purl.org/net/VideoGameOntology), use the following config file:

```yaml
vocabularies:
    - http://vocab.linkeddata.es/vgo/GameOntologyv3.owl # The URL of the vocabulary definition

types:
    Session:
        vocabularyNamespace: http://purl.org/net/VideoGameOntology#

  # ...
```

## All Types, Resolve Types and Exclude

If you use multiple vocabularies, and you need to generate all types for some ones,
only generate types when they are used for some others and exclude some types,
you can do so with this kind of configuration:

```yaml
vocabularies:
    # Schema.org classes will only be generated when one of its type is used in the other vocabularies.
    - { uri: 'https://schema.org/version/latest/schemaorg-current-https.rdf', format: null, allTypes: false }
    - http://vocab.linkeddata.es/vgo/GameOntologyv3.owl

allTypes: true # Generate all types by default for vocabularies
resolveTypes: true # Resolve types in other vocabularies

types:
    GameEvent:
        exclude: true # Exclude the GameEvent type
```

## Using Labels for naming the resources

If you are using vocabularies that have adopted numerical IDs, such as [Wikidata](https://www.wikidata.org) or [OBO ontologies](https://obofoundry.org), you might prefer to use the labels for naming your types/resources of interest instead of reusing the ID part of their URI fragment identifier, which is the default behavior.

You can use one of the dedicated options available at each level of your configuration to achieve this: 'nameAllFromLabels' for the full config level and the vocabulary level, or 'nameFromLabel' at the types level.

The two sub-options 'language' and 'namingConvention' are available for each of these levels and allow you to determine the preferred language when using the labels for naming purpose (default: ‘en’) and the naming convention on which naming will be based (i.e., 'snake case', default: 'camel case').

Using labels can also be preferred for any vocabulary as long as labels are defined. If no label is found, the ID part of the URI fragment identifier will be used as with the default behavior.

Here are basic examples for the different levels of configuration:

**General level**
```yaml
vocabularies:
    - { uri: http://purl.obolibrary.org/obo/ro.owl, format: rdfxml }

allTypes: true # Generate all types by default for vocabularies
nameAllFromLabels: true # Make use of the label information when possible to define all entities names
namingConvention: snake case
language: en
```
**Vocabulary level**
```yaml
vocabularies:
    - { uri: http://purl.obolibrary.org/obo/ro.owl, format: rdfxml, nameAllFromLabels: true, language: en, namingConvention: camel case }
    - { uri: https://raw.githubusercontent.com/w3c/dxwg/gh-pages/dcat/rdf/dcat3.rdf, format: rdfxml, nameAllFromLabels: false }

allTypes: true # Generate all types by default for vocabularies
```
**Type level**
```yaml
vocabularies:
    - { uri: http://www.wikidata.org/entity/Q115634351.rdf }
    - { uri: https://raw.githubusercontent.com/w3c/dxwg/gh-pages/dcat/rdf/dcat3.rdf, format: rdfxml }

types:
    Q115634351:
        vocabularyNamespace: http://www.wikidata.org/entity/
        nameFromLabel: true
        namingConvention: camel case
        language: es
    CatalogRecord:
        vocabularyNamespace: http://www.w3.org/ns/dcat#
        nameFromLabel: true
        language: es
```


## Checking GoodRelation Compatibility

If the `checkIsGoodRelations` option is set to `true`, the generator will emit a warning if an encountered property is not
par of the [GoodRelations](https://www.heppnetz.de/projects/goodrelations/) schema.

This is useful when generating e-commerce data models.

## Author PHPDoc

Add a `@author` PHPDoc annotation to class DocBlock.

Example:

```yaml
author: "Kévin Dunglas <kevin@les-tilleuls.coop>"
```

## PHP File Header

Prepend all generated PHP files with a custom comment.

Example:

```yaml
header: |
    /*
     * This file is part of the Ecommerce package.
     *
     * (c) Kévin Dunglas <kevin@dunglas.fr>
     *
     * For the full copyright and license information, please view the LICENSE
     * file that was distributed with this source code.
     */
```

## Disabling Generators and Creating Custom Ones

By default, all generators except `DoctrineMongoDBAttributeGenerator` are enabled.
You can specify the list of generators to use with the `annotationGenerators` and `attributeGenerators` option.

Example (enabling only the PHPDoc generator):

```yaml
annotationGenerators:
    - ApiPlatform\SchemaGenerator\AnnotationGenerator\PhpDocAnnotationGenerator
attributeGenerators: []
```

You can write your own generators by implementing the `AnnotationGeneratorInterface` or `AttributeGeneratorInterface`.
The `AbstractAnnotationGenerator` or `AbstractAttributeGenerator` provides helper methods
useful when creating your own generators.

Enabling a custom attribute generator and the PHPDoc generator:

```yaml
annotationGenerators:
    - ApiPlatform\SchemaGenerator\AnnotationGenerator\PhpDocAnnotationGenerator
attributeGenerators
    - Acme\Generators\MyGenerator
```

## Full Configuration Reference

```yaml
openApi:
    file:                 null

# RDF vocabularies
vocabularies:

    # Prototype
    uri:

        # RDF vocabulary to use
        uri:                  ~ # Example: 'https://schema.org/version/latest/schemaorg-current-https.rdf'

        # RDF vocabulary format
        format:               null # Example: rdfxml

        # Generate all types for this vocabulary, even if an explicit configuration exists. If allTypes is enabled globally, it can be disabled for this particular vocabulary
        allTypes:             null

        # Make use of the label information when possible to define all entities names in this vocabulary
        nameAllFromLabels:    false

        # The language to use, with this vocabulary, among the labels when used as entities names, e.g., en
        language: null

        # The naming convention to use, with this vocabulary, when naming entities from their labels; possible values are "snake case" or "camel case"(default), classes have first charater in uppercase
        namingConvention: null

        # Attributes (merged with generated attributes)
        attributes:           []

# Namespace of the vocabulary to import
vocabularyNamespace:  'https://schema.org/' # Example: 'http://www.w3.org/ns/activitystreams#'

# Relations configuration
relations:

    # OWL relation URIs containing cardinality information in the GoodRelations format
    uris:                 # Example: 'https://archive.org/services/purl/goodrelations/v1.owl'

        # Default:
        - https://archive.org/services/purl/goodrelations/v1.owl

    # The default cardinality to use when it cannot be extracted
    defaultCardinality:   (1..1) # One of "(0..1)"; "(0..*)"; "(1..1)"; "(1..*)"; "(*..0)"; "(*..1)"; "(*..*)"

# Debug mode
debug:                false

# Use old API Platform attributes (API Platform < 2.7)
apiPlatformOldAttributes: false

# IDs configuration
id:

    # Automatically add an ID field to entities
    generate:             true

    # The ID generation strategy to use ("none" to not let the database generate IDs).
    generationStrategy:   auto # One of "auto"; "none"; "uuid"; "mongoid"

    # Is the ID writable? Only applicable if "generationStrategy" is "uuid".
    writable:             false

# Generate interfaces and use Doctrine's Resolve Target Entity feature
useInterface:         false

# Emit a warning if a property is not derived from GoodRelations
checkIsGoodRelations: false

# A license or any text to use as header of generated files
header:               null # Example: '// (c) Kévin Dunglas <dunglas@gmail.com>'

# PHP namespaces
namespaces:

    # The global namespace's prefix
    prefix:               null # Example: App\

    # The namespace of the generated entities
    entity:               App\Entity # Example: App\Entity

    # The namespace of the generated enumerations
    enum:                 App\Enum # Example: App\Enum

    # The namespace of the generated interfaces
    interface:            App\Model # Example: App\Model

# Custom uses (for instance if you use a custom attribute)
uses:

    # Prototype
    name:

        # Name of this use
        name:                 ~ # Example: App\Attributes\MyAttribute

        # The alias to use for this use
        alias:                null

# Doctrine
doctrine:

    # Use Doctrine's ArrayCollection instead of standard arrays
    useCollection:        true

    # The Resolve Target Entity Listener config file path
    resolveTargetEntityConfigPath: null

    # The Resolve Target Entity Listener config file type
    resolveTargetEntityConfigType: XML # One of "XML"; "yaml"

    # Doctrine inheritance attributes (if set, no other attributes are generated)
    inheritanceAttributes: []

    # The inheritance type to use when an entity is referenced by another and has child
    inheritanceType:      JOINED # One of "JOINED"; "SINGLE_TABLE"; "SINGLE_COLLECTION"; "TABLE_PER_CLASS"; "COLLECTION_PER_CLASS"; "NONE"

    # Maximum length of any given database identifier, like tables or column names
    maxIdentifierLength:  63

# Symfony Validator Component
validator:

    # Generate @Assert\Type annotation
    assertType:           false

# The value of the phpDoc's @author annotation
author:               false # Example: 'Kévin Dunglas <dunglas@gmail.com>'

# Visibility of entities fields
fieldVisibility:      private # One of "private"; "protected"; "public"

# Set this flag to false to not generate getter, setter, adder and remover methods
accessorMethods:      true

# Set this flag to true to generate fluent setter, adder and remover methods
fluentMutatorMethods: false
rangeMapping:

    # Prototype
    name:                 ~

# Generate all types, even if an explicit configuration exists
allTypes:             false

# If a type is present in a vocabulary but not explicitly imported (types) or if the vocabulary is not totally imported (allTypes), it will be generated
resolveTypes:         false

# Make use of the label information when possible to define all entities names in all vocabularies
nameAllFromLabels:    false

# The language to use, with all vocabularies, among the labels when used as entities names, e.g., en
language: null

# The naming convention to use, with all vocabularies, when naming entities from their labels; possible values are "snake case" or "camel case"(default), classes have first charater in uppercase
namingConvention: null

# Types to import from the vocabulary
types:

    # Prototype
    id:

        # Exclude this type, even if "allTypes" is set to true"
        exclude:              false

        # Namespace of the vocabulary of this type (defaults to the global "vocabularyNamespace" entry)
        vocabularyNamespace:  null # Example: 'http://www.w3.org/ns/activitystreams#'

        # Is the class abstract? (null to guess)
        abstract:             null

        # Is the class embeddable?
        embeddable:           false

        # Type namespaces
        namespaces:

            # The namespace for the generated class (override any other defined namespace)
            class:                null

            # The namespace for the generated interface (override any other defined namespace)
            interface:            null

        # Attributes (merged with generated attributes)
        attributes:           []

        # The parent class, set to false for a top level class
        parent:               false

        # If declaring a custom class, this will be the class from which properties type will be guessed
        guessFrom:            Thing

        # Operations for the class
        operations:           []

        # Make use of the label information when possible to define the entity name for the current type
        nameFromLabels:    false

        # The language to use, with the current type, among the labels when used as entities names, e.g., en
        language: null

        # The naming convention to use, with the current type, when naming entities from their labels; possible values are "snake case" or "camel case"(default), classes have first charater in uppercase
        namingConvention: null

        # Import all existing properties
        allProperties:        false

        # Properties of this type to use
        properties:

            # Prototype
            id:

                # Exclude this property, even if "allProperties" is set to true"
                exclude:              false

                # The property range
                range:                null # Example: Offer
                cardinality:          unknown # One of "(0..1)"; "(0..*)"; "(1..1)"; "(1..*)"; "(*..0)"; "(*..1)"; "(*..*)"; "unknown"

                # Symfony Serialization Groups
                groups:               []

                # The doctrine mapped by attribute
                mappedBy:             null # Example: partOfSeason

                # The doctrine inversed by attribute
                inversedBy:           null # Example: episodes

                # Is the property readable?
                readable:             true

                # Is the property writable?
                writable:             true

                # Is the property nullable? (if null, cardinality will be used: will be true if no cardinality found)
                nullable:             null

                # Is the property required?
                required:             true

                # The property unique
                unique:               false

                # Is the property embedded?
                embedded:             false

                # Attributes (merged with generated attributes)
                attributes:           []

# Annotation generators to use
annotationGenerators:

    # Default:
    - ApiPlatform\SchemaGenerator\AnnotationGenerator\PhpDocAnnotationGenerator

# Attribute generators to use
attributeGenerators:

    # Defaults:
    - ApiPlatform\SchemaGenerator\AttributeGenerator\DoctrineOrmAttributeGenerator
    - ApiPlatform\SchemaGenerator\AttributeGenerator\DoctrineOrmAssociationOverrideAttributeGenerator
    - ApiPlatform\SchemaGenerator\AttributeGenerator\ApiPlatformCoreAttributeGenerator
    - ApiPlatform\SchemaGenerator\AttributeGenerator\ConstraintAttributeGenerator
    - ApiPlatform\SchemaGenerator\AttributeGenerator\ConfigurationAttributeGenerator

# Directories for custom generator twig templates
generatorTemplates:   []
```

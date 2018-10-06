# Getting Started

## Installation

If you use [the official distribution of API Platform](../distribution/index.md), the Schema Generator is already installed as a development dependency of your project and can be invoked through Docker:

    $ docker-compose exec php vendor/bin/schema

The Schema Generator can also [be downloaded independently as a PHAR](https://github.com/api-platform/schema-generator/releases) or installed in an existing project using [Composer](https://getcomposer.org):

    $ composer require --dev api-platform/schema-generator

## Model Scaffolding

Start by browsing [Schema.org](https://schema.org) and pick types applicable to your application. The website provides
tons of schemas including (but not limited to) representations of people, organization, event, postal address, creative
work and e-commerce structures.
Then, write a simple YAML config file like the following.

Here we will generate a data model for an address book with these data:

- a [`Person`](http://schema.org/Person) which inherits from [`Thing`](http://schema.org/Thing)
- a [`PostalAddress`](http://schema.org/PostalAddress) which inherits from [`ContactPoint`](http://schema.org/ContactPoint), which inherits from [`StructuredValue`](http://schema.org/StructuredValue), etc.

```yaml
# api/config/schema.yaml
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

    $ vendor/bin/schema generate-types api/src/ api/config/schema.yaml
      
> Using docker: `$ docker-compose exec php vendor/bin/schema generate-types src/ config/schema.yaml`

The following classes will be generated:

```yaml
types:
    Person:
        properties:
            name: ~
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

```php
<?php

declare(strict_types=1);

namespace App\Entity;

use ApiPlatform\Core\Annotation\ApiProperty;
use ApiPlatform\Core\Annotation\ApiResource;
use Doctrine\ORM\Mapping as ORM;

/**
 * A person (alive, dead, undead, or fictional).
 *
 * @see http://schema.org/Person Documentation on Schema.org
 *
 * @ORM\Entity
 * @ApiResource(iri="http://schema.org/Person")
 */
class Person
{
    /**
     * @var int|null
     *
     * @ORM\Id
     * @ORM\GeneratedValue(strategy="AUTO")
     * @ORM\Column(type="integer")
     */
    private $id;

    /**
     * @var string|null Family name. In the U.S., the last name of an Person. This can be used along with givenName instead of the name property.
     *
     * @ORM\Column(type="text", nullable=true)
     * @ApiProperty(iri="http://schema.org/familyName")
     */
    private $familyName;

    /**
     * @var string|null Given name. In the U.S., the first name of a Person. This can be used along with familyName instead of the name property.
     *
     * @ORM\Column(type="text", nullable=true)
     * @ApiProperty(iri="http://schema.org/givenName")
     */
    private $givenName;

    /**
     * @var string|null an additional name for a Person, can be used for a middle name
     *
     * @ORM\Column(type="text", nullable=true)
     * @ApiProperty(iri="http://schema.org/additionalName")
     */
    private $additionalName;

    /**
     * @var PostalAddress|null physical address of the item
     *
     * @ORM\ManyToOne(targetEntity="App\Entity\PostalAddress")
     * @ApiProperty(iri="http://schema.org/address")
     */
    private $address;

    public function getId(): ?int
    {
        return $this->id;
    }

    public function setFamilyName(?string $familyName): void
    {
        $this->familyName = $familyName;
    }

    public function getFamilyName(): ?string
    {
        return $this->familyName;
    }

    public function setGivenName(?string $givenName): void
    {
        $this->givenName = $givenName;
    }

    public function getGivenName(): ?string
    {
        return $this->givenName;
    }

    public function setAdditionalName(?string $additionalName): void
    {
        $this->additionalName = $additionalName;
    }

    public function getAdditionalName(): ?string
    {
        return $this->additionalName;
    }

    public function setAddress(?PostalAddress $address): void
    {
        $this->address = $address;
    }

    public function getAddress(): ?PostalAddress
    {
        return $this->address;
    }
}
```

```php
<?php

declare(strict_types=1);

namespace App\Entity;

use ApiPlatform\Core\Annotation\ApiProperty;
use ApiPlatform\Core\Annotation\ApiResource;
use Doctrine\ORM\Mapping as ORM;

/**
 * The mailing address.
 *
 * @see http://schema.org/PostalAddress Documentation on Schema.org
 *
 * @ORM\Entity
 * @ApiResource(iri="http://schema.org/PostalAddress")
 */
class PostalAddress
{
    /**
     * @var int|null
     *
     * @ORM\Id
     * @ORM\GeneratedValue(strategy="AUTO")
     * @ORM\Column(type="integer")
     */
    private $id;

    /**
     * @var string|null The country. For example, USA. You can also provide the two-letter \[ISO 3166-1 alpha-2 country code\](http://en.wikipedia.org/wiki/ISO\_3166-1).
     *
     * @ORM\Column(type="text", nullable=true)
     * @ApiProperty(iri="http://schema.org/addressCountry")
     */
    private $addressCountry;

    /**
     * @var string|null The locality. For example, Mountain View.
     *
     * @ORM\Column(type="text", nullable=true)
     * @ApiProperty(iri="http://schema.org/addressLocality")
     */
    private $addressLocality;

    /**
     * @var string|null The region. For example, CA.
     *
     * @ORM\Column(type="text", nullable=true)
     * @ApiProperty(iri="http://schema.org/addressRegion")
     */
    private $addressRegion;

    /**
     * @var string|null the post office box number for PO box addresses
     *
     * @ORM\Column(type="text", nullable=true)
     * @ApiProperty(iri="http://schema.org/postOfficeBoxNumber")
     */
    private $postOfficeBoxNumber;

    /**
     * @var string|null The postal code. For example, 94043.
     *
     * @ORM\Column(type="text", nullable=true)
     * @ApiProperty(iri="http://schema.org/postalCode")
     */
    private $postalCode;

    /**
     * @var string|null The street address. For example, 1600 Amphitheatre Pkwy.
     *
     * @ORM\Column(type="text", nullable=true)
     * @ApiProperty(iri="http://schema.org/streetAddress")
     */
    private $streetAddress;

    public function getId(): ?int
    {
        return $this->id;
    }

    public function setAddressCountry(?string $addressCountry): void
    {
        $this->addressCountry = $addressCountry;
    }

    public function getAddressCountry(): ?string
    {
        return $this->addressCountry;
    }

    public function setAddressLocality(?string $addressLocality): void
    {
        $this->addressLocality = $addressLocality;
    }

    public function getAddressLocality(): ?string
    {
        return $this->addressLocality;
    }

    public function setAddressRegion(?string $addressRegion): void
    {
        $this->addressRegion = $addressRegion;
    }

    public function getAddressRegion(): ?string
    {
        return $this->addressRegion;
    }

    public function setPostOfficeBoxNumber(?string $postOfficeBoxNumber): void
    {
        $this->postOfficeBoxNumber = $postOfficeBoxNumber;
    }

    public function getPostOfficeBoxNumber(): ?string
    {
        return $this->postOfficeBoxNumber;
    }

    public function setPostalCode(?string $postalCode): void
    {
        $this->postalCode = $postalCode;
    }

    public function getPostalCode(): ?string
    {
        return $this->postalCode;
    }

    public function setStreetAddress(?string $streetAddress): void
    {
        $this->streetAddress = $streetAddress;
    }

    public function getStreetAddress(): ?string
    {
        return $this->streetAddress;
    }
}
```

```php
<?php

declare(strict_types=1);

namespace App\Entity;

use ApiPlatform\Core\Annotation\ApiProperty;
use ApiPlatform\Core\Annotation\ApiResource;
use Doctrine\ORM\Mapping as ORM;

/**
 * The most generic type of item.
 *
 * @see http://schema.org/Thing Documentation on Schema.org
 *
 * @ORM\Entity
 * @ApiResource(iri="http://schema.org/Thing")
 */
class Thing
{
    /**
     * @var int|null
     *
     * @ORM\Id
     * @ORM\GeneratedValue(strategy="AUTO")
     * @ORM\Column(type="integer")
     */
    private $id;

    /**
     * @var string|null the name of the item
     *
     * @ORM\Column(type="text", nullable=true)
     * @ApiProperty(iri="http://schema.org/name")
     */
    private $name;

    public function getId(): ?int
    {
        return $this->id;
    }

    public function setName(?string $name): void
    {
        $this->name = $name;
    }

    public function getName(): ?string
    {
        return $this->name;
    }
}
```

Note that the generator takes care of creating directories corresponding to the namespace structure.

Without configuration file, the tool will build the entire Schema.org vocabulary. If no properties are specified for a given
type, all its properties will be generated.

The generator also supports enumerations generation. For subclasses of [`Enumeration`](https://schema.org/Enumeration), the
generator will automatically create a class extending the Enum type provided by [myclabs/php-enum](https://github.com/myclabs/php-enum).
Don't forget to install this library in your project. Refer you to PHP Enum documentation to see how to use it. The Symfony
validation annotation generator automatically takes care of enumerations to validate choices values.

A config file generating an enum class:

```yaml
types:
    OfferItemCondition: ~ # The generator will automatically guess that OfferItemCondition is subclass of Enum
```

The related PHP class:

```php
<?php

declare(strict_types=1);

namespace App\Enum;

use MyCLabs\Enum\Enum;

/**
 * A list of possible conditions for the item.
 *
 * @see http://schema.org/OfferItemCondition Documentation on Schema.org
 */
class OfferItemCondition extends Enum
{
    /**
     * @var string DamagedCondition
     */
    const DAMAGED_CONDITION = 'http://schema.org/DamagedCondition';
    /**
     * @var string NewCondition
     */
    const NEW_CONDITION = 'http://schema.org/NewCondition';
    /**
     * @var string RefurbishedCondition
     */
    const REFURBISHED_CONDITION = 'http://schema.org/RefurbishedCondition';
    /**
     * @var string UsedCondition
     */
    const USED_CONDITION = 'http://schema.org/UsedCondition';
}
```

### Going Further

* Browse [the configuration documentation](configuration.md)

## Cardinality Extraction

The Cardinality Extractor is a standalone tool (also used internally by the generator) extracting a property's cardinality.
It uses [GoodRelations](http://www.heppnetz.de/projects/goodrelations/) data when available. Other cardinalities are
guessed using the property's comment.
When cardinality cannot be automatically extracted, it's value is set to `unknown`.

Usage:

    $ vendor/bin/schema extract-cardinalities

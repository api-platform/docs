# Getting Started

## Installation

If you use [the official distribution of API Platform](../distribution/index.md), the Schema Generator is already installed as a development dependency of your project and can be invoked through Docker:

    $ docker-compose exec app vendor/bin/schema

The Schema Generator can also [be downloaded independently as a PHAR](https://github.com/api-platform/schema-generator/releases) or installed in an existing project using [Composer](https://getcomposer.org):

    $ composer require --dev api-platform/schema-generator

## Model Scaffolding

Start by browsing [Schema.org](https://schema.org) and pick types applicable to your application. The website provides
tons of schemas including (but not limited to) representations of people, organization, event, postal address, creative
work and e-commerce structures.
Then, write a simple YAML config file like the following (here we will generate a data model for an address book):

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
            gender: ~
            address: ~
            birthDate: ~
            telephone: ~
            email: ~
            url: ~
            jobTitle: ~
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

The following classes will be generated:

```yaml
types:
    Person:
        properties:
            name: ~
            familyName: ~
            givenName: ~
            additionalName: ~
            gender: ~
            address: ~
            birthDate: ~
            telephone: ~
            email: ~
            url: ~
            jobTitle: ~
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
use Symfony\Component\Validator\Constraints as Assert;

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
     * @var string|null the name of the item
     *
     * @ORM\Column(type="text", nullable=true)
     * @ApiProperty(iri="http://schema.org/name")
     */
    private $name;

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
     * @var string|null Gender of the person. While http://schema.org/Male and http://schema.org/Female may be used, text strings are also acceptable for people who do not identify as a binary gender.
     *
     * @ORM\Column(type="text", nullable=true)
     * @ApiProperty(iri="http://schema.org/gender")
     */
    private $gender;

    /**
     * @var PostalAddress|null physical address of the item
     *
     * @ORM\ManyToOne(targetEntity="App\Entity\PostalAddress")
     * @ApiProperty(iri="http://schema.org/address")
     */
    private $address;

    /**
     * @var \DateTimeInterface|null date of birth
     *
     * @ORM\Column(type="date", nullable=true)
     * @ApiProperty(iri="http://schema.org/birthDate")
     * @Assert\Date
     */
    private $birthDate;

    /**
     * @var string|null the telephone number
     *
     * @ORM\Column(type="text", nullable=true)
     * @ApiProperty(iri="http://schema.org/telephone")
     */
    private $telephone;

    /**
     * @var string|null email address
     *
     * @ORM\Column(type="text", nullable=true)
     * @ApiProperty(iri="http://schema.org/email")
     * @Assert\Email
     */
    private $email;

    /**
     * @var string|null URL of the item
     *
     * @ORM\Column(type="text", nullable=true)
     * @ApiProperty(iri="http://schema.org/url")
     * @Assert\Url
     */
    private $url;

    /**
     * @var string|null the job title of the person (for example, Financial Manager)
     *
     * @ORM\Column(type="text", nullable=true)
     * @ApiProperty(iri="http://schema.org/jobTitle")
     */
    private $jobTitle;

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

    public function setGender(?string $gender): void
    {
        $this->gender = $gender;
    }

    public function getGender(): ?string
    {
        return $this->gender;
    }

    public function setAddress(?PostalAddress $address): void
    {
        $this->address = $address;
    }

    public function getAddress(): ?PostalAddress
    {
        return $this->address;
    }

    public function setBirthDate(?\DateTimeInterface $birthDate): void
    {
        $this->birthDate = $birthDate;
    }

    public function getBirthDate(): ?\DateTimeInterface
    {
        return $this->birthDate;
    }

    public function setTelephone(?string $telephone): void
    {
        $this->telephone = $telephone;
    }

    public function getTelephone(): ?string
    {
        return $this->telephone;
    }

    public function setEmail(?string $email): void
    {
        $this->email = $email;
    }

    public function getEmail(): ?string
    {
        return $this->email;
    }

    public function setUrl(?string $url): void
    {
        $this->url = $url;
    }

    public function getUrl(): ?string
    {
        return $this->url;
    }

    public function setJobTitle(?string $jobTitle): void
    {
        $this->jobTitle = $jobTitle;
    }

    public function getJobTitle(): ?string
    {
        return $this->jobTitle;
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

Previous chapter: [Introduction](index.md)

Next chapter: [Configuration](configuration.md)

# Getting Started

## Installation

If you use [the official distribution of API Platform](../distribution/index.md), the Schema Generator is already installed as a development dependency of your project and can be invoked through Docker:

    $ docker-compose exec app vendor/bin/schema

The Schema Generator can also [be downloaded independently as a PHAR](https://github.com/api-platform/schema-generator/releases) or installed in an existing project using [Composer](https://getcomposer.org):

    $ composer require --dev api-platform/schema-generator

## Model scaffolding

Start by browsing [Schema.org](https://schema.org) and pick types applicable to your application. The website provides
tons of schemas including (but not limited to) representations of people, organization, event, postal address, creative
work and e-commerce structures.
Then, write a simple YAML config file like the following (here we will generate a data model for an address book):

```yaml
# app/config/schema.yml

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

    $ docker-compose exec app vendor/bin/schema generate-types src/ app/config/schema.yml

The following classes will be generated:

```php
<?php

// src/AppBundle/Entity/Thing.php

namespace AppBundle\Entity;

use Doctrine\ORM\Mapping as ORM;
use Symfony\Component\Validator\Constraints as Assert;

/**
 * The most generic type of item.
 *
 * @see http://schema.org/Thing Documentation on Schema.org
 *
 * @ORM\MappedSuperclass
 */
abstract class Thing
{
    /**
     * @var string $name The name of the item.
     * @Assert\Type(type="string")
     * @ORM\Column(nullable=true)
     */
    private $name;

    /**
     * Sets name.
     *
     * @param  string $name
     * @return $this
     */
    public function setName($name)
    {
        $this->name = $name;

        return $this;
    }

    /**
     * Gets name.
     *
     * @return string
     */
    public function getName()
    {
        return $this->name;
    }
}

```

```php
<?php

// src/AppBundle/Entity/Person.php

namespace AppBundle\Entity;

use Doctrine\ORM\Mapping as ORM;
use Symfony\Component\Validator\Constraints as Assert;

/**
 * A person (alive, dead, undead, or fictional).
 *
 * @see http://schema.org/Person Documentation on Schema.org
 *
 * @ORM\Entity
 */
class Person extends Thing
{
    /**
     * @var integer
     * @ORM\Column(type="integer")
     * @ORM\Id
     * @ORM\GeneratedValue(strategy="AUTO")
     */
    private $id;

    /**
     * @var string An additional name for a Person, can be used for a middle name.
     * @Assert\Type(type="string")
     * @ORM\Column(nullable=true)
     */
    private $additionalName;

    /**
     * @var PostalAddress Physical address of the item.
     * @ORM\ManyToOne(targetEntity="PostalAddress")
     */
    private $address;
    /**
     * @var \DateTime Date of birth.
     * @Assert\Date
     * @ORM\Column(type="date", nullable=true)
     */
    private $birthDate;

    /**
     * @var string Email address.
     * @Assert\Email
     * @ORM\Column(nullable=true)
     */
    private $email;

    /**
     * @var string Family name. In the U.S., the last name of an Person. This can be used along with givenName instead of the name property.
     * @Assert\Type(type="string")
     * @ORM\Column(nullable=true)
     */
    private $familyName;

    /**
     * @var string Gender of the person.
     * @Assert\Type(type="string")
     * @ORM\Column(nullable=true)
     */
    private $gender;

    /**
     * @var string Given name. In the U.S., the first name of a Person. This can be used along with familyName instead of the name property.
     * @Assert\Type(type="string")
     * @ORM\Column(nullable=true)
     */
    private $givenName;

    /**
     * @var string The job title of the person (for example, Financial Manager).
     * @Assert\Type(type="string")
     * @ORM\Column(nullable=true)
     */
    private $jobTitle;

    /**
     * @var string The telephone number.
     * @Assert\Type(type="string")
     * @ORM\Column(nullable=true)
     */
    private $telephone;

    /**
     * Sets id.
     *
     * @param  integer $id
     * @return $this
     */
    public function setId($id)
    {
        $this->id = $id;

        return $this;
    }

    /**
     * Gets id.
     *
     * @return integer
     */
    public function getId()
    {
        return $this->id;
    }

    /**
     * Sets additionalName.
     *
     * @param  string $additionalName
     * @return $this
     */
    public function setAdditionalName($additionalName)
    {
        $this->additionalName = $additionalName;

        return $this;
    }

    /**
     * Gets additionalName.
     *
     * @return string
     */
    public function getAdditionalName()
    {
        return $this->additionalName;
    }

    /**
     * Sets address.
     *
     * @param  PostalAddress $address
     * @return $this
     */
    public function setAddress(PostalAddress $address)
    {
        $this->address = $address;

        return $this;
    }

    /**
     * Gets address.
     *
     * @return PostalAddress
     */
    public function getAddress()
    {
        return $this->address;
    }

    /**
     * Sets birthDate.
     *
     * @param  \DateTime $birthDate
     * @return $this
     */
    public function setBirthDate(\DateTime $birthDate)
    {
        $this->birthDate = $birthDate;

        return $this;
    }

    /**
     * Gets birthDate.
     *
     * @return \DateTime
     */
    public function getBirthDate()
    {
        return $this->birthDate;
    }

    /**
     * Sets email.
     *
     * @param  string $email
     * @return $this
     */
    public function setEmail($email)
    {
        $this->email = $email;

        return $this;
    }

    /**
     * Gets email.
     *
     * @return string
     */
    public function getEmail()
    {
        return $this->email;
    }

    /**
     * Sets familyName.
     *
     * @param  string $familyName
     * @return $this
     */
    public function setFamilyName($familyName)
    {
        $this->familyName = $familyName;

        return $this;
    }

    /**
     * Gets familyName.
     *
     * @return string
     */
    public function getFamilyName()
    {
        return $this->familyName;
    }

    /**
     * Sets gender.
     *
     * @param  string $gender
     * @return $this
     */
    public function setGender($gender)
    {
        $this->gender = $gender;

        return $this;
    }

    /**
     * Gets gender.
     *
     * @return string
     */
    public function getGender()
    {
        return $this->gender;
    }

    /**
     * Sets givenName.
     *
     * @param  string $givenName
     * @return $this
     */
    public function setGivenName($givenName)
    {
        $this->givenName = $givenName;

        return $this;
    }

    /**
     * Gets givenName.
     *
     * @return string
     */
    public function getGivenName()
    {
        return $this->givenName;
    }

    /**
     * Sets jobTitle.
     *
     * @param  string $jobTitle
     * @return $this
     */
    public function setJobTitle($jobTitle)
    {
        $this->jobTitle = $jobTitle;

        return $this;
    }

    /**
     * Gets jobTitle.
     *
     * @return string
     */
    public function getJobTitle()
    {
        return $this->jobTitle;
    }

    /**
     * Sets telephone.
     *
     * @param  string $telephone
     * @return $this
     */
    public function setTelephone($telephone)
    {
        $this->telephone = $telephone;

        return $this;
    }

    /**
     * Gets telephone.
     *
     * @return string
     */
    public function getTelephone()
    {
        return $this->telephone;
    }
}

```

```php
<?php

// src/AppBundle/Entity/PostalAddress.php

namespace AppBundle\Entity;

use Doctrine\ORM\Mapping as ORM;
use Symfony\Component\Validator\Constraints as Assert;

/**
 * The mailing address.
 *
 * @see http://schema.org/PostalAddress Documentation on Schema.org
 *
 * @ORM\Entity
 */
class PostalAddress
{
    /**
     * @var integer
     * @ORM\Column(type="integer")
     * @ORM\Id
     * @ORM\GeneratedValue(strategy="AUTO")
     */
    private $id;

    /**
     * @var string The country. For example, USA. You can also provide the two-letter [ISO 3166-1 alpha-2 country code](http://en.wikipedia.org/wiki/ISO_3166-1).
     * @Assert\Type(type="string")
     * @ORM\Column(nullable=true)
     */
    private $addressCountry;

    /**
     * @var string The locality. For example, Mountain View.
     * @Assert\Type(type="string")
     * @ORM\Column(nullable=true)
     */
    private $addressLocality;

    /**
     * @var string The region. For example, CA.
     * @Assert\Type(type="string")
     * @ORM\Column(nullable=true)
     */
    private $addressRegion;

    /**
     * @var string The postal code. For example, 94043.
     * @Assert\Type(type="string")
     * @ORM\Column(nullable=true)
     */
    private $postalCode;

    /**
     * @var string $postOfficeBoxNumber The post office box number for PO box addresses.
     * @Assert\Type(type="string")
     * @ORM\Column(nullable=true)
     */
    private $postOfficeBoxNumber;

    /**
     * @var string The street address. For example, 1600 Amphitheatre Pkwy.
     * @Assert\Type(type="string")
     * @ORM\Column(nullable=true)
     */
    private $streetAddress;

    /**
     * Sets id.
     *
     * @param  integer $id
     * @return $this
     */
    public function setId($id)
    {
        $this->id = $id;

        return $this;
    }

    /**
     * Gets id.
     *
     * @return integer
     */
    public function getId()
    {
        return $this->id;
    }

    /**
     * Sets addressCountry.
     *
     * @param  string $addressCountry
     * @return $this
     */
    public function setAddressCountry($addressCountry)
    {
        $this->addressCountry = $addressCountry;

        return $this;
    }

    /**
     * Gets addressCountry.
     *
     * @return string
     */
    public function getAddressCountry()
    {
        return $this->addressCountry;
    }

    /**
     * Sets addressLocality.
     *
     * @param  string $addressLocality
     * @return $this
     */
    public function setAddressLocality($addressLocality)
    {
        $this->addressLocality = $addressLocality;

        return $this;
    }

    /**
     * Gets addressLocality.
     *
     * @return string
     */
    public function getAddressLocality()
    {
        return $this->addressLocality;
    }

    /**
     * Sets addressRegion.
     *
     * @param  string $addressRegion
     * @return $this
     */
    public function setAddressRegion($addressRegion)
    {
        $this->addressRegion = $addressRegion;

        return $this;
    }

    /**
     * Gets addressRegion.
     *
     * @return string
     */
    public function getAddressRegion()
    {
        return $this->addressRegion;
    }

    /**
     * Sets postalCode.
     *
     * @param  string $postalCode
     * @return $this
     */
    public function setPostalCode($postalCode)
    {
        $this->postalCode = $postalCode;

        return $this;
    }

    /**
     * Gets postalCode.
     *
     * @return string
     */
    public function getPostalCode()
    {
        return $this->postalCode;
    }

    /**
     * Sets postOfficeBoxNumber.
     *
     * @param  string $postOfficeBoxNumber
     * @return $this
     */
    public function setPostOfficeBoxNumber($postOfficeBoxNumber)
    {
        $this->postOfficeBoxNumber = $postOfficeBoxNumber;

        return $this;
    }

    /**
     * Gets postOfficeBoxNumber.
     *
     * @return string
     */
    public function getPostOfficeBoxNumber()
    {
        return $this->postOfficeBoxNumber;
    }

    /**
     * Sets streetAddress.
     *
     * @param  string $streetAddress
     * @return $this
     */
    public function setStreetAddress($streetAddress)
    {
        $this->streetAddress = $streetAddress;

        return $this;
    }

    /**
     * Gets streetAddress.
     *
     * @return string
     */
    public function getStreetAddress()
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

// src/AppBundle/Enum/OfferItemCondition.php

namespace AppBundle\Enum;

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

### Enabling API Platform Core support

PHP Schema is able to use [annotations provided by the core library](distribution/index.md#creating-the-model) and
to map properties to corresponding types of the used [external vocabulary](../core/external-vocabularies.md).
This is useful if you plan to use your generated data model to power a REST API.

To enable this generator along with others, add the following lines to your PHP Schema configuration file:

```yaml
annotationGenerators:
    - ApiPlatform\SchemaGenerator\AnnotationGenerator\PhpDocAnnotationGenerator
    - ApiPlatform\SchemaGenerator\AnnotationGenerator\DoctrineOrmAnnotationGenerator
    - ApiPlatform\SchemaGenerator\AnnotationGenerator\ConstraintAnnotationGenerator
    - ApiPlatform\SchemaGenerator\AnnotationGenerator\ApiPlatformCoreAnnotationGenerator
```

### Going further

* Browse [the configuration documentation](configuration.md)

## Cardinality extraction

The Cardinality Extractor is a standalone tool (also used internally by the generator) extracting a property's cardinality.
It uses [GoodRelations](http://www.heppnetz.de/projects/goodrelations/) data when available. Other cardinalities are
guessed using the property's comment.
When cardinality cannot be automatically extracted, it's value is set to `unknown`.

Usage:

    $ docker-compose exec app vendor/bin/schema extract-cardinalities

Previous chapter: [Introduction](index.md)

Next chapter: [Configuration](configuration.md)

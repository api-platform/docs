# Using External Vocabularies

JSON-LD allows to define classes and properties of your API with open vocabularies such as [Schema.org](https://schema.org)
and [Good Relations](http://www.heppnetz.de/projects/goodrelations/).

API Platform Core provides annotations usable on PHP classes and properties for specifying a related external [IRI](https://en.wikipedia.org/wiki/Internationalized_resource_identifier).

```php
<?php
// api/src/Entity/Book.php
namespace App\Entity;

use ApiPlatform\Metadata\ApiProperty;
use ApiPlatform\Metadata\ApiResource;

#[ApiResource(types: ['http://schema.org/Book'])]
class Book
{
    // ...

    #[ApiProperty(types: ['http://schema.org/name'])]
    public $name;
    
    // ...
}
```

The generated JSON for products and the related context document will now use external IRIs according to the specified annotations:

`GET /books/22`

```json
{
  "@context": "/contexts/Book",
  "@id": "/books/22",
  "@type": "https://schema.org/Book",
  "name": "My awesome book"
}
```

`GET /contexts/Book`

```json
{
  "@context": {
    "@vocab": "http://example.com/apidoc#",
    "hydra": "http://www.w3.org/ns/hydra/core#",
    "name": "https://schema.org/name"
  }
}
```

An extended list of existing open vocabularies is available on [the Linked Open Vocabularies (LOV) database](http://lov.okfn.org/dataset/lov/).

By default, when using [validations](validation.md) API Platform Core will try to define known [Schema.org](https://schema.org) types as IRIs for your properties if you did not provide any in your `#[ApiProperty]` annotations.
Built-in mapping is:

Constraints                                          | Schema.org type                   |
---------------------------------------------------- |-----------------------------------|
`Symfony\Component\Validator\Constraints\Url`        | `http://schema.org/url`           |
`Symfony\Component\Validator\Constraints\Email`      | `http://schema.org/email`         |
`Symfony\Component\Validator\Constraints\Uuid`       | `http://schema.org/identifier`    |
`Symfony\Component\Validator\Constraints\CardScheme` | `http://schema.org/identifier`    |
`Symfony\Component\Validator\Constraints\Bic`        | `http://schema.org/identifier`    |
`Symfony\Component\Validator\Constraints\Iban`       | `http://schema.org/identifier`    |
`Symfony\Component\Validator\Constraints\Date`       | `http://schema.org/Date`          |
`Symfony\Component\Validator\Constraints\DateTime`   | `http://schema.org/DateTime`      |
`Symfony\Component\Validator\Constraints\Time`       | `http://schema.org/Time`          |
`Symfony\Component\Validator\Constraints\Image`      | `http://schema.org/image`         |
`Symfony\Component\Validator\Constraints\File`       | `http://schema.org/MediaObject`   |
`Symfony\Component\Validator\Constraints\Currency`   | `http://schema.org/priceCurrency` |
`Symfony\Component\Validator\Constraints\Isbn`       | `http://schema.org/isbn`          |
`Symfony\Component\Validator\Constraints\Issn`       | `http://schema.org/issn`          |

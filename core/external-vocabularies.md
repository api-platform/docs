# Using External Vocabularies

JSON-LD allows to define classes and properties of your API with open vocabularies such as
[Schema.org](https://schema.org) and
[Good Relations](https://www.heppnetz.de/projects/goodrelations/).

API Platform provides attributes usable on PHP classes and properties for specifying a related
external [IRI](https://en.wikipedia.org/wiki/Internationalized_resource_identifier).

```php
<?php
// api/src/ApiResource/Book.php with Symfony or app/ApiResource/Book.php with Laravel
namespace App\ApiResource;

use ApiPlatform\Metadata\ApiProperty;
use ApiPlatform\Metadata\ApiResource;

#[ApiResource(types: ['https://schema.org/Book'])]
class Book
{
    // ...

    #[ApiProperty(iris: ['https://schema.org/name'])]
    public $name;

    // ...
}
```

The generated JSON for products and the related context document will now use external IRIs
according to the specified attributes:

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

## Resource `types`, Property `iris`, and Property `types`

These three attributes each control a different output, and mixing them up is a common mistake:

- `types` on `#[ApiResource]` sets the RDF class of the resource. It replaces the JSON-LD `@type`
  and sets the `subClassOf` value in the Hydra documentation.
- `iris` on `#[ApiProperty]` sets the RDF property IRI. It is used in the JSON-LD `@context` (as
  shown above) and in the Hydra documentation. Without it, API Platform builds a local term from
  the property name instead.
- `types` on `#[ApiProperty]` produces no RDF output at all. Its only effect is to populate the
  `externalDocs.url` field of that property in the generated OpenAPI schema.

To map both the class and a property to Schema.org, use `types` on the resource and `iris` on the
property:

```php
#[ApiResource(types: ['https://schema.org/Store'])]
class Store
{
    #[ApiProperty(iris: ['https://schema.org/name'])]
    public string $name;
}
```

An extended list of existing open vocabularies is available on
[the Linked Open Vocabularies (LOV) database](https://lov.linkeddata.es/dataset/lov/).

By default, when using [validations](validation.md) API Platform will try to define known
[Schema.org](https://schema.org) types as IRIs for your properties if you did not provide any in
your `#[ApiProperty]` attributes.

- Symfony built-in mapping is:

    | Constraints                                          | Schema.org type                    |
    | ---------------------------------------------------- | ---------------------------------- |
    | `Symfony\Component\Validator\Constraints\Url`        | `https://schema.org/url`           |
    | `Symfony\Component\Validator\Constraints\Email`      | `https://schema.org/email`         |
    | `Symfony\Component\Validator\Constraints\Uuid`       | `https://schema.org/identifier`    |
    | `Symfony\Component\Validator\Constraints\CardScheme` | `https://schema.org/identifier`    |
    | `Symfony\Component\Validator\Constraints\Bic`        | `https://schema.org/identifier`    |
    | `Symfony\Component\Validator\Constraints\Iban`       | `https://schema.org/identifier`    |
    | `Symfony\Component\Validator\Constraints\Date`       | `https://schema.org/Date`          |
    | `Symfony\Component\Validator\Constraints\DateTime`   | `https://schema.org/DateTime`      |
    | `Symfony\Component\Validator\Constraints\Time`       | `https://schema.org/Time`          |
    | `Symfony\Component\Validator\Constraints\Image`      | `https://schema.org/image`         |
    | `Symfony\Component\Validator\Constraints\File`       | `https://schema.org/MediaObject`   |
    | `Symfony\Component\Validator\Constraints\Currency`   | `https://schema.org/priceCurrency` |
    | `Symfony\Component\Validator\Constraints\Isbn`       | `https://schema.org/isbn`          |
    | `Symfony\Component\Validator\Constraints\Issn`       | `https://schema.org/issn`          |

- Laravel built-in mapping is:

    | Laravel Validation Rule   | Schema.org Type                  |
    | ------------------------- | -------------------------------- |
    | `url`                     | `https://schema.org/url`         |
    | `email`                   | `https://schema.org/email`       |
    | `uuid`                    | `https://schema.org/identifier`  |
    | `date`                    | `https://schema.org/Date`        |
    | `date_format:Y-m-d H:i:s` | `https://schema.org/DateTime`    |
    | `date_format:H:i`         | `https://schema.org/Time`        |
    | `image`                   | `https://schema.org/image`       |
    | `mimes` or `mimetypes`    | `https://schema.org/MediaObject` |

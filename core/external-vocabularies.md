# Using External Vocabularies

JSON-LD allows to define classes and properties of your API with open vocabularies such as [Schema.org](https://schema.org)
and [Good Relations](http://www.heppnetz.de/projects/goodrelations/).

API Platform Core provides annotations usable on PHP classes and properties for specifying a related external [IRI](http://en.wikipedia.org/wiki/Internationalized_resource_identifier).

```php
<?php
// src/AppBundle/Entity/Book.php

namespace AppBundle\Entity;

use ApiPlatform\Core\Annotation\ApiProperty;
use ApiPlatform\Core\Annotation\ApiResource;

/**
 * @ApiResource(iri="http://schema.org/Book")
 */
class Book
{
    // ...

    /**
     * ...
     * @ApiProperty(iri="http://schema.org/name")
     */
    public $name;
}
```

The generated JSON for products and the related context document will now use external IRIs according to the specified annotations:

`GET /books/22`

```json
{
  "@context": "/contexts/Book",
  "@id": "/books/22",
  "@type": "https://schema.org/Product",
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

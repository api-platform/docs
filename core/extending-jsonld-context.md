# Extending JSON-LD AND Hydra Contexts

## JSON-LD

<p align="center" class="symfonycasts"><a href="https://symfonycasts.com/screencast/api-platform/json-ld?cid=apip"><img src="../distribution/images/symfonycasts-player.png" alt="JSON-LD screencast"><br>Watch the JSON-LD screencast</a>

API Platform provides the possibility to extend the JSON-LD context of properties. This allows you to describe JSON-LD-typed
values, inverse properties using the `@reverse` keyword and you can even overwrite the `@id` property this way. Everything you define
within the following annotation will be passed to the context. This provides a generic way to extend the context.

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

    #[ApiProperty(
        types: ['http://schema.org/name'],
        jsonldContext: [
            '@id' => 'http://yourcustomid.com',
            '@type' => 'http://www.w3.org/2001/XMLSchema#string',
            'someProperty' => [
                'a' => 'textA',
                'b' => 'textB'
            ]
        ]
    )]
    public $name;
    
    // ...
}
```

The generated context will now have your custom attributes set:

`GET /contexts/Book`

```json
{
  "@context": {
    "@vocab": "http://example.com/apidoc#",
    "hydra": "http://www.w3.org/ns/hydra/core#",
    "name": {
      "@id": "http://yourcustomid.com",
      "@type": "http://www.w3.org/2001/XMLSchema#string",
      "someProperty": {
        "a": "textA",
        "b": "textB"
      }
    }
  }
}
```

Note that you do not have to provide the `@id` attribute. If you do not provide an `@id` attribute, the value from `iri` will be used.

## Hydra

<p align="center" class="symfonycasts"><a href="https://symfonycasts.com/screencast/api-platform/hydra?cid=apip"><img src="../distribution/images/symfonycasts-player.png" alt="Hydra screencast"><br>Watch the Hydra screencast</a></p>

It's also possible to replace the Hydra context used by the documentation generator:

[codeSelector]

```php
<?php
// api/src/Entity/Book.php
namespace App\Entity;

use ApiPlatform\Metadata\ApiResource;
use ApiPlatform\Metadata\Get;

#[ApiResource(operations: [
  new Get(hydraContext: ['foo' => 'bar'])
])]
class Book
{
    //...
}
```

```yaml
# api/config/api_platform/resources.yaml
App\Entity\Book:
    operations:
        ApiPlatform\Metadata\Get:
            hydraContext: { foo: 'bar' }
```

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<!-- api/config/api_platform/resources.xml -->

<resources xmlns="https://api-platform.com/schema/metadata/resources"
           xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
           xsi:schemaLocation="https://api-platform.com/schema/metadata/resources
           https://api-platform.com/schema/metadata/resources.xsd">
    <resource class="App\Entity\Book">
        <operations>
            <operation class="ApiPlatform\Metadata\Get">              
                <hydraContext>
                    <values>
                        <value name="foo">bar</value>
                    </values>
                </hydraContext>
            </operation>
        </operations>
    </resource>
</resources>
```

[/codeSelector]

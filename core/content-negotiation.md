# Content Negotiation

The API system has built-in [content negotiation](https://en.wikipedia.org/wiki/Content_negotiation) capabilities.
It leverages the [`willdurand/negotiation`](https://github.com/willdurand/Negotiation) library.

By default, only the [JSON-LD](https://json-ld.org) format is enabled. However API Platform Core supports many more formats and can be extended.

The framework natively supports JSON-LD, GraphQL, JSONAPI, HAL, raw JSON, XML, YAML and CSV (YAML and CSV support is only available if you use Symfony 3.2+).

Both XML and JSON formats are experimental and there are no assurance that we will not break them.

API Platform Core will automatically detect the best resolving format depending on:

* enabled formats (see below)
* the requested format, specified in either the `Accept` HTTP header or as an extension appended to the URL

Available formats are:

Format                                                          | Format name  | MIME types                    | Backward Compatibility guaranteed
----------------------------------------------------------------|--------------|-------------------------------|----------------------------------------
[JSON-LD](https://json-ld.org)                                  | `jsonld`     | `application/ld+json`         | yes
[GraphQL](graphql.md)                                           | n/a          | n/a                           | yes
[JSONAPI](http://jsonapi.org/)                                  | `jsonapi`    | `application/vnd.api+json`    | yes
[HAL](http://stateless.co/hal_specification.html)               | `jsonhal`    | `application/hal+json`        | yes
[JSON](https://www.json.org/)                                   | `json`       |  `application/json`           | no
[XML](https://www.w3.org/XML/)                                  | `xml`        | `application/xml`, `text/xml` | no
[YAML](http://yaml.org/)                                        | `yaml`       | `application/x-yaml`          | no
[CSV](https://tools.ietf.org/html/rfc4180)                      | `csv`        | `text/csv`                    | no
[HTML](https://whatwg.org/) (API docs)                          | `html`       | `text/html`                   | no

If the client requested format is not specified (if it's not supported, it will throw an HTTP bad request error), the response format will be the first format defined in the `formats` configuration key (see below).
An example using the built-in XML support is available in [Behat specs](https://github.com/api-platform/core/blob/master/features/main/content_negotiation.feature).

The API Platform content negotiation system is extendable. Support for other formats can be added by [creating and registering appropriate encoders and, sometimes, normalizers](https://symfony.com/doc/current/serializer.html#adding-normalizers-and-encoders). Adding support for other
standard hypermedia formats upstream is welcome. Don't hesitate to contribute by adding your encoders and normalizers
to API Platform Core.

## Enabling Several Formats

The first required step is to configure allowed formats. The following configuration will enable the support of XML (built-in)
and of a custom format called `myformat` and having `application/vnd.myformat` as [MIME type](https://en.wikipedia.org/wiki/Media_type).

```yaml
# api/config/packages/api_platform.yaml
api_platform:
    # ...

    formats:
        jsonld:   ['application/ld+json']
        jsonhal:  ['application/hal+json']
        jsonapi:  ['application/vnd.api+json']
        json:     ['application/json']
        xml:      ['application/xml', 'text/xml']
        yaml:     ['application/x-yaml']
        csv:      ['text/csv']
        html:     ['text/html']
        myformat: ['application/vnd.myformat']
```

To enable GraphQL support, [read the dedicated chapter](graphql.md).

Because the Symfony Serializer component is able to serialize objects in XML, sending an `Accept` HTTP header with the
`text/xml` string as value is enough to retrieve XML documents from our API. However API Platform knows nothing about the
`myformat` format. We need to register an encoder and optionally a normalizer for this format.


## Enabling Additional Formats On a Specific Resource/Operation 

Support for specific formats can also be added at resource or operation level, using the `formats` attribute.

```php
<?php
// api/src/Entity/Book.php

namespace App\Entity;

/**
 * @ApiResource(attributes={"formats"={"xml", "jsonld", "csv"={"text/csv"}}})
 */
class Book
{
    // ...
}
```

In the example above, `xml` or `jsonld` will be allowed and there is no need to specify the mime types as they are already defined in the configuration.
Additionally the `csv` format is added with the mime type `text/csv`.

It is also important to notice that the usage of this attribute will override the formats defined in the configuration, therefore
this configuration might disable the `json` or the `html` on this resource for example.

You can specify different accepted formats at operation level too:
```php
<?php
// api/src/Entity/Book.php

namespace App\Entity;

/**
 * @ApiResource(
*      collectionOperations={"get"={"formats"={"xml"={"text/xml"}}}},
*      attributes={"formats"={"jsonld", "csv"={"text/csv"}}}
*  )
 */
class Book
{
    // ...
}
```

As an alternative to annotations, you can also use XML or YAML, the example above would become:

XML:
```xml
<resources xmlns="https://api-platform.com/schema/metadata"
           xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
           xsi:schemaLocation="https://api-platform.com/schema/metadata
           https://api-platform.com/schema/metadata/metadata-2.0.xsd">
    <resource class="App\Entity\Greeting">
        <collectionOperations>
            <collectionOperation name="get">
                <attribute name="formats">
                    <attribute name="xml">
                        <attribute>text/xml</attribute>
                    </attribute>
                    <!-- works also with <attribute name="xml">text/xml</attribute> -->
                </attribute>
            </collectionOperation>
        </collectionOperations>

        <attribute name="formats">
            <attribute>jsonld</attribute> <!-- format already defined in the config -->
            <attribute name="csv">text/csv</attribute>
        </attribute>
    </resource>
</resources>
```
YAML:
```yaml
resources:
    App\Entity\Book:
        collectionOperations:
            get:
                formats:
                    xml: ['text/xml'] # works also with "text/html"
        attributes:
            formats:
               0: 'jsonld' # format already defined in the config
               csv: 'text/csv'
```

## Registering a Custom Serializer

If you are adding support for a format not supported by default by API Platform nor by the Symfony Serializer Component,
you need to create custom encoder, decoder and eventually a normalizer and a denormalizer. Refer to the
Symfony documentation to learn [how to create and register such classes](https://symfony.com/doc/current/cookbook/serializer.html#adding-normalizers-and-encoders).

API Platform Core will automatically call the serializer with your defined format name (`myformat` in previous examples)
as `format` parameter during the deserialization process. Then it will return the result to the client with the asked MIME
type using its built-in responder.

## Writing a Custom Normalizer

Using composition is the recommended way to implement a custom normalizer. You can use the following template to start with your
own implementation of `CustomItemNormalizer`:

```yaml
# api/config/services.yaml
services:
    'App\Serializer\CustomItemNormalizer':
        arguments: [ '@api_platform.serializer.normalizer.item' ]
        # Uncomment if you don't use the autoconfigure feature
        #tags: [ 'serializer.normalizer' ]
    
    # ...
```

```php
<?php
// api/src/Serializer/CustomItemNormalizer.php

namespace App\Serializer;

use Symfony\Component\Serializer\Normalizer\DenormalizerInterface;
use Symfony\Component\Serializer\Normalizer\NormalizerInterface;

final class CustomItemNormalizer implements NormalizerInterface, DenormalizerInterface
{
    private $normalizer;

    public function __construct(NormalizerInterface $normalizer)
    {
        if (!$normalizer instanceof DenormalizerInterface) {
            throw new \InvalidArgumentException('The normalizer must implement the DenormalizerInterface');
        }

        $this->normalizer = $normalizer;
    }

    public function denormalize($data, $class, $format = null, array $context = [])
    {
        return $this->normalizer->denormalize($data, $class, $format, $context);
    }

    public function supportsDenormalization($data, $type, $format = null)
    {
        return $this->normalizer->supportsDenormalization($data, $type, $format);
    }

    public function normalize($object, $format = null, array $context = [])
    {
        return $this->normalizer->normalize($object, $format, $context);
    }

    public function supportsNormalization($data, $format = null)
    {
        return $this->normalizer->supportsNormalization($data, $format);
    }
}
```

For example if you want to make the `csv` format work for even complex entities with a lot of hierarchy, you have to
flatten or remove too complex relations:

```php
<?php
// api/src/Serializer/CustomItemNormalizer.php

namespace App\Serializer;

use Symfony\Component\Serializer\Normalizer\DenormalizerInterface;
use Symfony\Component\Serializer\Normalizer\NormalizerInterface;

class CustomItemNormalizer implements NormalizerInterface, DenormalizerInterface
{
    // ...

    public function normalize($object, $format = null, array $context = [])
    {
        $result = $this->normalizer->normalize($object, $format, $context);

        if ('csv' !== $format || !is_array($result)) {
            return $result;
        }

        foreach ($result as $key => $value) {
            if (is_array($value) && array_keys(array_keys($value)) === array_keys($value)) {
                unset($result[$key]);
            }
        }

        return $result;
    }

    // ...
}
```

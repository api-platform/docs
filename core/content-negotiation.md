# Content Negotiation

The API system has built-in [content negotiation](https://en.wikipedia.org/wiki/Content_negotiation) capabilities.

By default, only the [JSON-LD](https://json-ld.org) and JSON formats are enabled. However API Platform Core supports many more formats and can be extended.

The framework natively supports JSON-LD (and Hydra), GraphQL, JSON:API, HAL, YAML, CSV, HTML (API docs), raw JSON and raw XML.
Using the raw JSON or raw XML formats is discouraged, prefer using JSON-LD instead, which provides more feature and is as easy to use.

API Platform also supports [JSON Merge Patch (RFC 7396)](https://tools.ietf.org/html/rfc7396) the JSON:API [`PATCH`](https://tools.ietf.org/html/rfc5789) formats, as well as [Problem Details (RFC 7807)](https://tools.ietf.org/html/rfc7807), Hydra and JSON:API error formats.

<p align="center" class="symfonycasts"><a href="https://symfonycasts.com/screencast/api-platform/formats?cid=apip"><img src="../distribution/images/symfonycasts-player.png" alt="Formats screencast"><br>Watch the Formats screencast</a></p>

API Platform Core will automatically detect the best resolving format depending on:

* enabled formats (see below)
* the requested format, specified in either [the `Accept` HTTP header](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Accept) or as an extension appended to the URL

Available formats are:

Format                                                          | Format name  | MIME types                    | Backward Compatibility guaranteed
----------------------------------------------------------------|--------------|-------------------------------|----------------------------------------
[JSON-LD](https://json-ld.org)                                  | `jsonld`     | `application/ld+json`         | yes
[GraphQL](graphql.md)                                           | n/a          | n/a                           | yes
[JSON:API](http://jsonapi.org/)                                 | `jsonapi`    | `application/vnd.api+json`    | yes
[HAL](http://stateless.co/hal_specification.html)               | `jsonhal`    | `application/hal+json`        | yes
[YAML](http://yaml.org/)                                        | `yaml`       | `application/x-yaml`          | no
[CSV](https://tools.ietf.org/html/rfc4180)                      | `csv`        | `text/csv`                    | no
[HTML](https://whatwg.org/) (API docs)                          | `html`       | `text/html`                   | no
[XML](https://www.w3.org/XML/)                                  | `xml`        | `application/xml`, `text/xml` | no
[JSON](https://www.json.org/)                                   | `json`       |  `application/json`           | no

If the client's requested format is not specified, the response format will be the first format defined in the `formats` configuration key (see below).
If the request format is not supported, an [Unsupported Media Type](https://developer.mozilla.org/fr/docs/Web/HTTP/Status/415) error will be returned.

Examples showcasing how to use the different mechanisms are available [in the API Platform test suite](https://github.com/api-platform/core/blob/main/features/main/content_negotiation.feature).

## Configuring Formats Globally

The first required step is to configure allowed formats. The following configuration will enable the support of XML (built-in)
and of a custom format called `myformat` and having `application/vnd.myformat` as [MIME type](https://en.wikipedia.org/wiki/Media_type).

```yaml
# config/packages/api_platform.yaml
api_platform:
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

## Configuring PATCH Formats

By default, API Platform supports JSON Merge Patch and JSON:API PATCH formats.
Support for the JSON:API PATCH format is automatically enabled if JSON:API support is enabled.
JSON Merge Patch support must be enabled explicitly:

```yaml
# config/packages/api_platform.yaml
api_platform:
    patch_formats:
        json:     ['application/merge-patch+json']
        jsonapi:  ['application/vnd.api+json']
```

When support for at least one PATCH format is enabled, [an `Accept-Patch` HTTP header](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Accept-Patch) containing the list of supported patch formats is automatically added to all HTTP responses for items.

## Configuring Error Formats

API Platform will try to send to the client the error format matching with the format request with the `Accept` HTTP headers (or the URL extension). For instance, if a client request a JSON-LD representation of a resource, and an error occurs, then API Platform will serialize this error using the Hydra format (Hydra is a vocabulary for JSON-LD containing a standard representation of API errors).

Available formats can also be configured:

```yaml
# config/packages/api_platform.yaml
api_platform:
    error_formats:
        jsonproblem:                   ['application/problem+json']
        jsonld:                        ['application/ld+json']      # Hydra error formats
        jsonapi:                       ['application/vnd.api+json']
```

## Configuring Formats For a Specific Resource or Operation

Support for specific formats can also be configured at resource and operation level using the `input_formats` and `output_formats` attributes.
`input_formats` controls the formats accepted in request bodies while `output_formats` controls formats available for responses.

The `formats` attribute can be used as a shortcut, it sets both the `input_formats` and `output_formats` in one time.

```php
<?php
// api/src/Entity/Book.php

namespace App\Entity;

use ApiPlatform\Core\Annotation\ApiResource;

 #[ApiResource(formats: ['xml', 'jsonld', 'csv' => ['text/csv']])]
class Book
{
    // ...
}
```

In the example above, `xml` or `jsonld` will be allowed and there is no need to specify the MIME types as they are already defined in the configuration.
Additionally the `csv` format is added with the MIME type `text/csv`.

It is also important to notice that the usage of this attribute will override the formats defined in the configuration, therefore
this configuration might disable the `json` or the `html` on this resource for example.

You can specify different accepted formats at operation level too, it's especially convenient for to configure formats available for the `PATCH` method:

<code-selector>

```php
<?php
// api/src/Entity/Book.php

namespace App\Entity;

use ApiPlatform\Core\Annotation\ApiResource;

#[ApiResource(
    formats: ['jsonld', 'csv' => ['text/csv']],
    itemOperations: [
        'patch' => [
            'input_formats' => [
                'json' => ['application/merge-patch+json'],
            ],
        ],
    ],
)]
class Book
{
    // ...
}
```

```yaml
resources:
    App\Entity\Book:
        attributes:
            formats:
               0: 'jsonld' # format already defined in the config
               csv: 'text/csv'
        itemOperations:
            get:
                formats:
                    json: ['application/merge-patch+json'] # works also with "application/merge-patch+json"
```

```xml
<resources xmlns="https://api-platform.com/schema/metadata"
           xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
           xsi:schemaLocation="https://api-platform.com/schema/metadata
           https://api-platform.com/schema/metadata/metadata-2.0.xsd">
    <resource class="App\Entity\Greeting">
        <attribute name="formats">
            <attribute>jsonld</attribute> <!-- format already defined in the config -->
            <attribute name="csv">text/csv</attribute>
        </attribute>

        <itemOperations>
            <itemOperation name="get">
                <attribute name="input_formats">
                    <attribute name="json">
                        <attribute>application/merge-patch+json</attribute>
                    </attribute>
                    <!-- works also with <attribute name="json">application/merge-patch+json</attribute> -->
                </attribute>
            </itemOperation>
        </itemOperations>
    </resource>
</resources>
```

</code-selector>

## Supporting Custom Formats

The API Platform content negotiation system is extendable.
You can add support for formats not available by default by creating custom normalizer and encoders.
Refer to the Symfony documentation to learn [how to create and register such classes](https://symfony.com/doc/current/serializer.html#adding-normalizers-and-encoders).

Then, register the new format in the configuration:

```yaml
# config/packages/api_platform.yaml
api_platform:
    formats:
        # ...
        myformat: ['application/vnd.myformat']
```

API Platform Core will automatically call the serializer with your defined format name as `format` parameter during the deserialization process (`myformat` in the example).
It will then return the result to the client with the requested MIME type using its built-in responder.
For non-standard formats, [a vendor, vanity or unregistered MIME type should be used](https://en.wikipedia.org/wiki/Media_type#Vendor_tree).

### Reusing the API Platform Infrastructure

Using composition is the recommended way to implement a custom normalizer. You can use the following template to start your
own implementation of `CustomItemNormalizer`:

```yaml
# config/services.yaml
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
flatten or remove overly complex relations:

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

### Contributing Support for New Formats

Adding support for **standard** formats upstream is welcome!
We'll be glad to merge new encoders and normalizers in API Platform Core.

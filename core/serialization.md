# The Serialization Process

## Overall Process

API Platform embraces and extends the Symfony Serializer Component to transform PHP entities in hypermedia API responses.

The main serialization process has two stages:

![Serializer workflow](images/SerializerWorkflow.png)

> As you can see in the picture above, an array is used as a man in the middle. This way, Encoders will only deal with turning specific formats into arrays and vice versa. The same way, Normalizers will deal with turning specific objects into arrays and vice versa. 
-- [The Symfony documentation](https://symfony.com/doc/current/components/serializer.html)

Unlike Symfony itself, API Platform leverages custom normalizers, its router and the [data provider](data-providers.md) system to do an advanced tranformation. Metadata are added to the generated document including links, type information, pagination data or available filters.

The API Platform Serializer is very extensible, you can register custom normalizers and encoders to support other formats. You can also decorate existing normalizers to customize their behaviors.

## Available Serializers

### Normalizers

* [JSON-LD](https://json-ld.org) serializer
`api_platform.jsonld.normalizer.item`

JSON-LD, or JavaScript Object Notation for Linked Data, is a method of encoding Linked Data using JSON. It is a World Wide Web Consortium Recommendation.

* [HAL](https://en.wikipedia.org/wiki/Hypertext_Application_Language) serializer
`api_platform.hal.normalizer.item`

* JSON, XML, CSV, YAML serializer (using the Symfony serializer)
`api_platform.serializer.normalizer.item`

## Decorating a Serializer and Add Extra Data
In the following example, we will see how we add extra informations to the output.
Here is how we add the date on each request in `GET`:

```yaml
# app/config/services.yml

services:

  # ...

    'AppBundle\Serializer\ApiNormalizer':
        decorates: 'api_platform.jsonld.normalizer.item'
        arguments: [ '@AppBundle\Serializer\ApiNormalizer.inner' ]
        autoconfigure: false
```

```php
<?php

// src/Appbundle/Serializer/ApiSerializer

namespace AppBundle\Serializer;

use ApiPlatform\Core\Serializer\AbstractItemNormalizer;
use Symfony\Component\Console\Exception\InvalidArgumentException;
use Symfony\Component\Serializer\Normalizer\NormalizerInterface;

final class ApiNormalizer extends AbstractItemNormalizer
{
    public function __construct(NormalizerInterface $decorated)
    {
        $this->decorated = $decorated;
    }
    
    public function supportsNormalization($data, $format = null)
    {
        return $this->decorated->supportsNormalization($data, $format);
    }
    
    public function normalize($object, $format = null, array $context = [])
    {
        $data = $this->decorated->normalize($object, $format, $context);
        if (is_array($data)) {
            $data['date'] = date(\DateTime::RFC3339);
        }
        return $data;
    }
    
    public function supportsDenormalization($data, $type, $format = null)
    {
        return $this->decorated->supportsNormalization($data, $type, $format);
    }
    
    public function denormalize($data, $class, $format = null, array $context = [])
    {
        return $this->decorated->denormalise($data, $class, $format, $context);
    }
}
```

Previous chapter: [Swagger Support](core/swagger.md)

Next chapter: [Schema Generator: Introduction](schema-generator/index.md)

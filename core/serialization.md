# The Serialization Process

## Overall process

API Platform embraces and extends the Symfony Serializer Component to transform PHP entities in hypermedia API responses.

The main serialization process has two stages:

![Serializer workflow](images/SerializerWorkflow.png)

> As you can see in the picture above, an array is used as a man in the middle. This way, Encoders will only deal with turning specific formats into arrays and vice versa. The same way, Normalizers will deal with turning specific objects into arrays and vice versa.

[The Symfony documentation](https://symfony.com/doc/current/components/serializer.html)

Unlike Symfony itself, API Platform leverages custom normalizers, its router and the [data provider](data-providers.md) system to do an advanced tranformation. Metadata are added to the generated document including links, type information, pagination data or available filters.

The API Platform Serializer is very extensible, you can register custom normalizers and encoders to support other formats. You can also decorate existing normalizers to customize their behaviors.

## Available Serializers

### Normalizers

* [JSON-LD](https://json-ld.org) serializer
`api_platform.jsonld.normalizer.item`

JSON-LD, or JavaScript Object Notation for Linked Data, is a method of encoding Linked Data using JSON. It is a World Wide Web Consortium Recommendation.

* [HAL](https://en.wikipedia.org/wiki/Hypertext_Application_Language) serializer
`api_platform.hal.normalizer.item`

* JSON, XML, CSV serializer (using the Symfony serializer)
`api_platform.serializer.normalizer.item`

## Decorates a Serializer and Add Extra Data
In the following example, we will see how we add extra informations to the output.
Here is how we add the date on each request in `GET`:

```yaml
 # app/config/services.yml

services:
        app.custom.jsonld.normalizer:
        class: AppBundle\Serializer\ApiNormalizer
        autowire: true
        arguments: [ '@api_platform.metadata.resource.metadata_factory', '@api_platform.metadata.property.name_collection_factory', '@api_platform.metadata.property.metadata_factory', '@api_platform.iri_converter', '@api_platform.resource_class_resolver', '@api_platform.jsonld.context_builder', '@api_platform.property_accessor', '@api_platform.name_converter', '@serializer.mapping.class_metadata_factory','@app.collection.order']
        tags:
            - { name: serializer.normalizer, priority: 12 }
```

```php
<?php

// src/Appbundle/Serializer/ApiSerializer

namespace AppBundle\Serializer;

use ApiPlatform\Core\Api\IriConverterInterface;
use ApiPlatform\Core\Api\ResourceClassResolverInterface;
use ApiPlatform\Core\Exception\InvalidArgumentException;
use ApiPlatform\Core\JsonLd\ContextBuilderInterface;
use ApiPlatform\Core\JsonLd\Serializer\JsonLdContextTrait;
use ApiPlatform\Core\Metadata\Property\Factory\PropertyMetadataFactoryInterface;
use ApiPlatform\Core\Metadata\Property\Factory\PropertyNameCollectionFactoryInterface;
use ApiPlatform\Core\Metadata\Resource\Factory\ResourceMetadataFactoryInterface;
use ApiPlatform\Core\Serializer\AbstractItemNormalizer;
use ApiPlatform\Core\Serializer\ContextTrait;
use Symfony\Component\PropertyAccess\PropertyAccessorInterface;
use Symfony\Component\Serializer\Mapping\Factory\ClassMetadataFactoryInterface;
use Symfony\Component\Serializer\NameConverter\NameConverterInterface;

final class ApiNormalizer extends AbstractItemNormalizer
{
    use ContextTrait;
    use JsonLdContextTrait;

    const FORMAT = 'jsonld';

    private $resourceMetadataFactory;
    private $contextBuilder;

    public function __construct(ResourceMetadataFactoryInterface $resourceMetadataFactory, PropertyNameCollectionFactoryInterface $propertyNameCollectionFactory, PropertyMetadataFactoryInterface $propertyMetadataFactory, IriConverterInterface $iriConverter, ResourceClassResolverInterface $resourceClassResolver, ContextBuilderInterface $contextBuilder, PropertyAccessorInterface $propertyAccessor = null, NameConverterInterface $nameConverter = null, ClassMetadataFactoryInterface $classMetadataFactory = null)
    {
        parent::__construct($propertyNameCollectionFactory, $propertyMetadataFactory, $iriConverter, $resourceClassResolver, $propertyAccessor, $nameConverter, $classMetadataFactory);

        $this->resourceMetadataFactory = $resourceMetadataFactory;
        $this->contextBuilder = $contextBuilder;
    }

    public function supportsNormalization($data, $format = null)
    {
        return self::FORMAT === $format && parent::supportsNormalization($data, $format);
    }

    public function normalize($object, $format = null, array $context = [])
    {
        $resourceClass = $this->resourceClassResolver->getResourceClass($object, $context['resource_class'] ?? null, true);
        $resourceMetadata = $this->resourceMetadataFactory->create($resourceClass);
        $data = $this->addJsonLdContext($this->contextBuilder, $resourceClass, $context);

        // Use resolved resource class instead of given resource class to support multiple inheritance child types
        $context['resource_class'] = $resourceClass;

        $rawData = parent::normalize($object, $format, $context);
        if (!is_array($rawData)) {
            return $rawData;
        }

       $data['@id'] = $this->iriConverter->getIriFromItem($object);
       $data['@type'] = $resourceMetadata->getIri() ?: $resourceMetadata->getShortName();

		 // e.g. Here we add for each normalization the current date
       $extra =  new \DateTime('now');
       $rawData['date'] =  $extra->format('Y-m-d');

       return $data + $rawData;
    }


    public function supportsDenormalization($data, $type, $format = null)
    {
        return self::FORMAT === $format && parent::supportsDenormalization($data, $type, $format);
    }

    public function denormalize($data, $class, $format = null, array $context = [])
    {
       //
    }
}
```

# Content Negotiation

The API system has builtin [content negotiation](https://en.wikipedia.org/wiki/Content_negotiation) capabilities.
It leverages the [`willdurand/negotiation`](https://github.com/willdurand/Negotiation) library.

The only supported format by default is [JSON-LD](https://json-ld.org). Support for other formats such as XML or [Protobuf](https://developers.google.com/protocol-buffers/)
can be added easily.

API Platform Core will automatically detect the best format to return according to the `Accept` HTTP header sent by the
client and enabled formats. If no format asked by the client is supported by the server, the response will be sent in the
first format defined in the `support_formats` configuration key (see below).

An example using the builtin XML serializer is available in Behat specs: https://github.com/api-platform/core/blob/master/features/content_negotiation.feature

## Enabling Several Formats

The first required step is to configure allowed formats. The following configuration will enable the support of a format
called `myformat` and having `application/vnd.myformat` as [MIME type](https://en.wikipedia.org/wiki/Media_type).

```yaml
# app/config/config.yml

api_platform:
    # ...
    supported_formats:
        jsonld:                        ['application/ld+json']
        myformat:                      ['application/vnd.myformat']
```

## Registering a Custom Format in the Negotiation Library

If the format you want to use is not supported by default in the Negotiation library, you must register it using a [compiler
pass](https://symfony.com/doc/current/components/dependency_injection/compilation.html#creating-a-compiler-pass):

```php
// src/AppBundle/DependencyInjection/Compiler/MyFormatPass.php

namespace AppBundle\DependencyInjection\Compiler;

use Symfony\Component\DependencyInjection\ContainerBuilder;
use Symfony\Component\DependencyInjection\Compiler\CompilerPassInterface;

final class MyFormatPass implements CompilerPassInterface
{
    public function process(ContainerBuilder $container)
    {
        // ...

        $container->getDefinition('api.format_negotiator')->addMethodCall('registerFormat', [
            'myformat', ['application/vnd.myformat'], true,
        ]);
    }
}
```

Don't forget to register your compiler pass into the container from the `Bundle::build(ContainerBuilder $container)` method of your bundle:

```php
// src/AppBundle/AppBundle.php

namespace AppBundle;

use AppBundle\DependencyInjection\Compiler\MyFormatPass;
use Symfony\Component\DependencyInjection\ContainerBuilder;
use Symfony\Component\HttpKernel\Bundle\Bundle;

final class AppBundle extends Bundle
{
    public function build(ContainerBuilder $container)
    {
        $container->addCompilerPass(new MyFormatPass());
    }
}
```

## Registering a Custom Serializer

Then you need to create custom encoder, decoder and eventually a normalizer and a denormalizer for your format. API Platform
Core relies on the Symfony Serializer Component. [Refer to its dedicated documentation](https://symfony.com/doc/current/cookbook/serializer.html#adding-normalizers-and-encoders)
to learn how to create and register such classes.

API Platform Core will automatically call the serializer with your defined format name (`myformat` in previous examples)
as `format` parameter during the deserialization process.

## Creating a responder

Finally, you need to create a class that will convert the raw data to formatted data and the according HTTP response.
Here is an example responder using the XML serializer shipped with the Symfony Serializer Component:

```php
// src/AppBundle/EventListener/XmlResponderViewListener.php

namespace AppBundle\EventListener;

use ApiPlatform\Core\Metadata\Resource\Factory\ResourceMetadataFactoryInterface;
use Symfony\Component\HttpFoundation\Request;
use Symfony\Component\HttpFoundation\Response;
use Symfony\Component\HttpKernel\Event\GetResponseForControllerResultEvent;
use Symfony\Component\Serializer\SerializerInterface;

/**
 * Serializes data in XML then builds the response object.
 */
final class XmlResponderViewListener
{
    const FORMAT = 'xml';

    private $serializer;
    private $resourceMetadataFactory;

    public function __construct(SerializerInterface $serializer, ResourceMetadataFactoryInterface $resourceMetadataFactory)
    {
        $this->serializer = $serializer;
        $this->resourceMetadataFactory = $resourceMetadataFactory;
    }

    /**
     * In an API context, converts any data to a XML response.
     *
     * @param GetResponseForControllerResultEvent $event
     *
     * @return Response|mixed
     */
    public function onKernelView(GetResponseForControllerResultEvent $event)
    {
        $controllerResult = $event->getControllerResult();

        if ($controllerResult instanceof Response) {
            return $controllerResult;
        }

        $request = $event->getRequest();

        $format = $request->attributes->get('_api_format');
        if (self::FORMAT !== $format) {
            return $controllerResult;
        }

        switch ($request->getMethod()) {
            case Request::METHOD_POST:
                $status = 201;
                break;

            case Request::METHOD_DELETE:
                $status = 204;
                break;

            default:
                $status = 200;
                break;
        }

        $resourceClass = $request->attributes->get('_resource_class');
        $resourceMetadata = $this->resourceMetadataFactory->create($resourceClass);
        $context = $resourceMetadata->getAttribute('normalization_context', []);

        $response = new Response(
            $this->serializer->serialize($controllerResult, self::FORMAT, $context),
            $status,
            ['Content-Type' => 'application/xml']
        );

        $event->setResponse($response);
    }
}
```

The last step is to register the event listener on [the `kernel.view` event](http://symfony.com/doc/current/components/http_kernel/introduction.html#the-kernel-view-event)
dispatched by Symfony:

```yaml
# app/config/services.yml

services:
    # ...

    app.xml_responder_view_listener:
        class:     'AppBundle\EventListener\XmlResponderViewListener'
        arguments: [ '@api_platform.serializer', '@api_platform.metadata.resource.metadata_factory' ]
        tags:
            - { name: 'kernel.event_listener', event: 'kernel.view', method: 'onKernelView' }
```

Previous chapter: [Using external (JSON-LD) vocabularies](external-vocabularies.md)<br>
Next chapter: [Security](security.md)

# Accept `application/x-www-form-urlencoded` Form Data

API Platform only supports raw documents as request input (encoded in JSON, XML, YAML...). This has many advantages including support of types and the ability to send back to the API documents originally retrieved through a `GET` request.
But sometimes - for instance, to support legacy clients - it is necessary to accept inputs encoded in the traditional [`application/x-www-form-urlencoded`](https://www.w3.org/TR/html401/interact/forms.html#h-17.13.4.1) format (HTML form content type). This can easily be done using [the powerful event system](events.md) of the framework.

**⚠ Important warning:** adding support for `application/x-www-form-urlencoded` make your API vulnerable to [CSRF attacks](https://www.owasp.org/index.php/Cross-Site_Request_Forgery_(CSRF)). Be sure to enable [proper countermeasures](https://github.com/dunglas/DunglasAngularCsrfBundle).

In this tutorial, we will decorate the default `DeserializeListener` class to handle form data if applicable, and delegate to the built-in listener for other cases.

## Create your `DeserializeListener` Decorator

This decorator is able to denormalize posted form data to the target object. In case of other format, it fallbacks to the original [DeserializeListener](https://github.com/api-platform/core/blob/91dc2a4d6eeb79ea8dec26b41e800827336beb1a/src/Bridge/Symfony/Bundle/Resources/config/api.xml#L85-L91).

```php
<?php

// src/AppBundle/EventListener/DeserializeListener.php

namespace AppBundle\EventListener;

use ApiPlatform\Core\Exception\RuntimeException;
use ApiPlatform\Core\Util\RequestAttributesExtractor;
use Symfony\Component\HttpFoundation\Request;
use Symfony\Component\HttpKernel\Event\GetResponseEvent;
use ApiPlatform\Core\EventListener\DeserializeListener as DecoratedListener;
use Symfony\Component\Serializer\Normalizer\DenormalizerInterface;
use ApiPlatform\Core\Serializer\SerializerContextBuilderInterface;

final class DeserializeListener
{
    private $decorated;
    private $denormalizer;
    private $serializerContextBuilder;

    public function __construct(DenormalizerInterface $denormalizer, SerializerContextBuilderInterface $serializerContextBuilder, DecoratedListener $decorated)
    {
        $this->denormalizer = $denormalizer;
        $this->serializerContextBuilder = $serializerContextBuilder;
        $this->decorated = $decorated;
    }

    public function onKernelRequest(GetResponseEvent $event) {
        $request = $event->getRequest();
        if ($request->isMethodSafe() || $request->isMethod(Request::METHOD_DELETE)) {
            return;
        }

        if ('form' === $request->getContentType()) {
            $this->denormalizeFormRequest($request);
        } else {
            $this->decorated->onKernelRequest($event);
        }
    }

    private function denormalizeFormRequest(Request $request)
    {
        try {
            $attributes = RequestAttributesExtractor::extractAttributes($request);
        } catch (RuntimeException $e) {
            return;
        }
        $context = $this->serializerContextBuilder->createFromRequest($request, false, $attributes);
        $populated = $request->attributes->get('data');
        if (null !== $populated) {
            $context['object_to_populate'] = $populated;
        }
        
        $data = $request->request->all();
        $object = $this->denormalizer->denormalize($data, $attributes['resource_class'], null, $context);
        $request->attributes->set('data', $object);
    }
}
```

## Create the Service Definition

```yaml
# app/config/services.yml

services:
    # ...
    app.listener.decorating_deserialize:
        class: 'AppBundle\EventListener\DeserializeListener'
        arguments: ['@api_platform.serializer', '@api_platform.serializer.context_builder', '@api_platform.listener.request.deserialize']
        tags:
            - { name: 'kernel.event_listener', event: 'kernel.request', method: 'onKernelRequest', priority: 2 }
```

## Cleanup the Original Listener

The decorated DeserializeListener is called on demand, so it's better to eliminate its own tags:

```php
<?php

// src/AppBundle/AppBundle.php

namespace AppBundle;

use Symfony\Component\DependencyInjection\Compiler\CompilerPassInterface;
use Symfony\Component\DependencyInjection\ContainerBuilder;
use Symfony\Component\HttpKernel\Bundle\Bundle;

class AppBundle extends Bundle
{
    public function build(ContainerBuilder $container)
    {
        parent::build($container);
        $container->addCompilerPass(new class implements CompilerPassInterface {
            public function process(ContainerBuilder $container) {
                $container
                    ->findDefinition('api_platform.listener.request.deserialize');
                    ->clearTags();
            }
        });
    }
}

```

Previous chapter: [Operation Path Naming](operation-path-naming.md)

Next chapter: [FOSUserBundle Integration](fosuser-bundle.md)

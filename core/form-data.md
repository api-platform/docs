# Accept `application/x-www-form-urlencoded` Form Data

API Platform only supports raw documents as request input (encoded in JSON, XML, YAML...). This has many advantages including support of types and the ability to send back to the API documents originally retrieved through a `GET` request.
However, sometimes - for instance, to support legacy clients - it is necessary to accept inputs encoded in the traditional [`application/x-www-form-urlencoded`](https://www.w3.org/TR/html401/interact/forms.html#h-17.13.4.1) format (HTML form content type). This can easily be done using [the powerful event system](events.md) of the framework.

**⚠ Adding support for `application/x-www-form-urlencoded` makes your API vulnerable to [CSRF attacks](https://www.owasp.org/index.php/Cross-Site_Request_Forgery_(CSRF)). Be sure to enable proper countermeasures [such as DunglasAngularCsrfBundle](https://github.com/dunglas/DunglasAngularCsrfBundle).**

In this tutorial, we will decorate the default `DeserializeListener` class to handle form data if applicable, and delegate to the built-in listener for other cases.

## Create your `DeserializeListener` Decorator

This decorator is able to denormalize posted form data to the target object. In case of other format, it fallbacks to the original [DeserializeListener](https://github.com/api-platform/core/blob/91dc2a4d6eeb79ea8dec26b41e800827336beb1a/src/Bridge/Symfony/Bundle/Resources/config/api.xml#L85-L91).

```php
<?php
// api/src/EventListener/DeserializeListener.php

namespace App\EventListener;

use ApiPlatform\Core\Util\RequestAttributesExtractor;
use Symfony\Component\HttpFoundation\Request;
use Symfony\Component\HttpKernel\Event\RequestEvent;
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

    public function onKernelRequest(RequestEvent $event): void {
        $request = $event->getRequest();
        if ($request->isMethodCacheable(false) || $request->isMethod(Request::METHOD_DELETE)) {
            return;
        }

        if ('form' === $request->getContentType()) {
            $this->denormalizeFormRequest($request);
        } else {
            $this->decorated->onKernelRequest($event);
        }
    }

    private function denormalizeFormRequest(Request $request): void
    {
        if (!$attributes = RequestAttributesExtractor::extractAttributes($request)) {
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

## Creating the Service Definition

```yaml
# config/services.yaml
services:
    # ...
    'App\EventListener\DeserializeListener':
        tags:
            - { name: 'kernel.event_listener', event: 'kernel.request', method: 'onKernelRequest', priority: 2 }
        # Autoconfiguration must be disabled to set a custom priority
        autoconfigure: false
        decorates: 'api_platform.listener.request.deserialize'
        arguments:
            $decorated: '@App\EventListener\DeserializeListener.inner'
```

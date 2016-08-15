#How to accept posted form data

###Create your DeserializeListener decorator:

This decorator is able to denormalize posted form data to the target object. In case of other format, it fallbacks to the original DeserializeListener.

```php
namespace AppBundle\Listener;

use ApiPlatform\Core\Exception\RuntimeException;
use ApiPlatform\Core\Util\RequestAttributesExtractor;
use Symfony\Component\HttpFoundation\Request;
use Symfony\Component\HttpKernel\Event\GetResponseEvent;
use ApiPlatform\Core\EventListener\DeserializeListener as DecoratedListener;
use Symfony\Component\Serializer\Normalizer\DenormalizerInterface;
use ApiPlatform\Core\Serializer\SerializerContextBuilderInterface;

class DeserializeListener
{
    /** @var DecoratedListener */
    private $decorated;

    /** @var DenormalizerInterface */
    private $denormalizer;

    /** @var SerializerContextBuilderInterface */
    private $serializerContextBuilder;


    public function __construct(DecoratedListener $decorated, DenormalizerInterface $denormalizer, SerializerContextBuilderInterface $serializerContextBuilder)
    {
        $this->decorated = $decorated;
        $this->denormalizer = $denormalizer;
        $this->serializerContextBuilder = $serializerContextBuilder;
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
        $data = $request->request->all();
        $object = $this->denormalizer ->denormalize($data, $attributes['resource_class'], null, $context);
        $request->attributes->set('data', $object);
    }
}
```

###Then, create the service definition:

```yml
    app.listener.request.deserialize:
        class: AppBundle\Listener\DeserializeListener
        arguments: ["@api_platform.listener.request.deserialize", "@api_platform.serializer", "@api_platform.serializer.context_builder"]
        tags:
        - { name: kernel.event_listener, event: kernel.request, method: onKernelRequest, priority: 2 }
```

###Clean up the original listener

The decorated [DeserializeListener](https://github.com/api-platform/core/blob/91dc2a4d6eeb79ea8dec26b41e800827336beb1a/src/Bridge/Symfony/Bundle/Resources/config/api.xml#L85-L91)
is called on demand, so it's better to eliminate its own tags:

```php
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
                $listener = $container->findDefinition('api_platform.listener.request.deserialize');
                $listener->clearTags();
            }
        });
    }
}

```

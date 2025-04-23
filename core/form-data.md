# Accept `application/x-www-form-urlencoded` Form Data

API Platform only supports raw documents as request input (encoded in JSON, XML, YAML...). This has many advantages
including support of types and the ability to send back to the API documents originally retrieved through a `GET` request.
However, sometimes - for instance, to support legacy clients - it is necessary to accept inputs encoded in the traditional
[`application/x-www-form-urlencoded`](https://www.w3.org/TR/html401/interact/forms.html#h-17.13.4.1) format
(HTML form content type). This can easily be done using the powerful [System providers and processors](extending.md#system-providers-and-processors)
of the framework.

> [!WARNING]  
> Adding support for `application/x-www-form-urlencoded` makes your API vulnerable to [CSRF (Cross-Site Request Forgery)](https://www.owasp.org/index.php/Cross-Site_Request_Forgery_(CSRF)) attacks.  
> It's crucial to implement proper countermeasures to protect your application.
>
> If you're using Symfony, make sure you enable [Stateless CSRF protection](https://symfony.com/blog/new-in-symfony-7-2-stateless-csrf).  
>
> If you're working with Laravel, refer to the [Laravel CSRF documentation](https://laravel.com/docs/csrf) to ensure
> adequate protection against such attacks.

In this tutorial, we will decorate the default `DeserializeListener` class to handle form data if applicable, and delegate to the built-in listener for other cases.

## Create your `FormRequestProcessorDecorator` processor

This decorator is able to denormalize posted form data to the target object. In case of other format, it fallbacks to the original [DeserializeListener](https://github.com/api-platform/core/blob/91dc2a4d6eeb79ea8dec26b41e800827336beb1a/src/Bridge/Symfony/Bundle/Resources/config/api.xml#L85-L91).

```php
<?php
// api/src/State/FormRequestProcessorDecorator.php using Symfony or app/State/FormRequestProcessorDecorator.php using Laravel

namespace App\State;

use ApiPlatform\State\ProcessorInterface;
use Symfony\Component\HttpFoundation\Request;
use ApiPlatform\Metadata\Operation;
use ApiPlatform\Serializer\SerializerContextBuilderInterface;
use Symfony\Component\Serializer\Normalizer\DenormalizerInterface;

final class FormRequestProcessorDecorator implements ProcessorInterface
{
    public function __construct(
        private readonly ProcessorInterface $decorated,
        private readonly DenormalizerInterface $denormalizer,
        private readonly SerializerContextBuilderInterface $serializerContextBuilder
    ) {}

    public function process(mixed $data, Operation $operation, array $uriVariables = [], array $context = []): mixed
    {
        // If the content type is form data, we process it separately
        if ('form' === $data->getContentType()) {
            return $this->handleFormRequest($data);
        }

        // Delegate the processing to the original processor for other cases
        return $this->decorated->process($data, $operation, $uriVariables, $context);
    }

    /**
     * Handle form requests by deserializing the data into the correct entity
     */
    private function handleFormRequest(Request $request)
    {
        $attributes = $request->attributes->get('_api_attributes');
        if (!$attributes) {
            return null;
        }

        $context = $this->serializerContextBuilder->createFromRequest($request, false, $attributes);

        // Deserialize the form data into an entity
        $data = $request->request->all();
        
        return $this->denormalizer->denormalize($data, 'App\Entity\SomeEntity', null, $context);
    }
}
```

Next, configure the `FormRequestProcessorDecorator` according to whether you're using Symfony or Laravel, as shown below:

### Creating the Service Definition using Symfony

```yaml
# api/config/services.yaml
services:
  # ...
    App\State\FormRequestProcessorDecorator:
        decorates: api_platform.state.processor
        arguments:
            $decorated: '@App\State\FormRequestProcessorDecorator.inner'
            $denormalizer: '@serializer'
            $serializerContextBuilder: '@api_platform.serializer.context_builder'
        tags:
            - { name: 'api_platform.state.processor' }
```

### Registering a Decorated Processor using Laravel

```php
<?php
// app/Providers/AppServiceProvider.php

namespace App\Providers;

use Illuminate\Support\ServiceProvider;
use App\State\FormRequestProcessorDecorator;
use ApiPlatform\Core\State\ProcessorInterface;
use Symfony\Component\Serializer\Normalizer\DenormalizerInterface;
use ApiPlatform\Core\Serializer\SerializerContextBuilderInterface;

class AppServiceProvider extends ServiceProvider
{
    public function register()
    {
        $this->app->bind(ProcessorInterface::class, function ($app) {
            $decoratedProcessor = $app->make(ProcessorInterface::class); 

            return new FormRequestProcessorDecorator(
                $decoratedProcessor,
                $app->make(DenormalizerInterface::class),
                $app->make(SerializerContextBuilderInterface::class)
            );
        });
    }
}
```

## Using your `FormRequestProcessorDecorator` processor

Finally, you can use the processor in your API Resource like this:

```php
<?php
// api/src/ApiResource/SomeEntity.php with Symfony or app/ApiResource/SomeEntity.php with Laravel

namespace App\ApiResource;

use ApiPlatform\Metadata\Post;
use App\State\FormRequestProcessorDecorator;

#[Post(processor: FormRequestProcessorDecorator::class)]
class SomeEntity
{
    //...
}
```
